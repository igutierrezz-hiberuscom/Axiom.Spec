# Increment: Cross-cutting e2e sweep + adversarial edge-case review

Status: closed
Date: 2026-07-24

## Goal

This is **INC-X1**, the FINAL item (17 of 17) of the Axiom evolution batch
(`giggly-imagining-moore.md`, "Transversal" / "INC-X1 — Barrido e2e + review
de edge-cases end-to-end"). All 16 prior increments (Clusters A, T, S, W) are
closed and were independently verified; the full monorepo suite was green
throughout. This increment does not ship new product behavior; it:

1. Chains as much of the shipped batch as is hermetically feasible in ONE
   flow (a new e2e test file), proving the increments actually COMPOSE
   end-to-end, not just in isolation.
2. Performs an adversarial review across the batch hunting for real
   integration-level defects, edge cases, and cross-increment
   inconsistencies — explicitly NOT restating what the 16 increments'
   own green tests already prove.

## Context

Read first: `Axiom.SDD/AGENTS.md` (canonical lifecycle), the full plan
(`giggly-imagining-moore.md`), and all 16 prior increment specs under
`Axiom.Spec/specs/increments/INC-20260724-*/README.md` (Cluster A: topology,
adopt-creates-axiom-repo, provenance-lifecycle-manifest, export-eject-rollback,
unified-axiom-mcp; Cluster T: cmm-replaces-graphify-codegraph,
rtk-skill-invoked, concision-skills-policy; Cluster S:
autoskills-lock-hygiene; Cluster W: git-worktree-services,
worktree-isolation-execution, worktree-provisioning,
worktree-provider-isolation, worktree-harvest-cleanup,
worktree-mode-selection, sdd-artifact-freshness).

Each of the 16 increments has extensive unit/integration/e2e coverage of its
OWN scope, but several genuine cross-increment SEAMS were only covered
per-increment with narrow fixtures that deliberately avoided combining with
sibling increments (e.g., every existing worktree e2e test either never runs
`axiom configure` first, or sidesteps `harvestAndCleanupExecution`'s default
`force:false` by explicitly passing `force:true`). This increment's job is to
walk those seams deliberately.

## Scope

- New e2e test file: `apps/cli/tests/e2e/cross-cutting-batch.e2e.test.ts`
  (3 tests — see "Result" below for exactly what each chains/reproduces).
- Adversarial review of the shipped code across the batch (no code scope
  beyond trivial fixes — see Non-goals).
- This spec file.

## Non-goals

- Re-implementing or expanding any of the 16 shipped increments' own scope.
- Fixing any of the review's CONFIRMED findings beyond trivial, obviously-
  correct one-liners (none were found — every finding below requires a real
  design decision, e.g. which flag to add, which `repoRoot` to default to,
  left to the batch orchestrator's triage, per this increment's own brief).
- Migrating Axiom's own repos (`Axiom.SDD`/`Axiom.Spec`) to the new model —
  out of scope for the whole plan.
- Integration into `Axiom.Spec/specs/00_Resumen_Ejecutivo.md …
  08_Glosario.md` — per the brief's explicit instruction, the orchestrator
  performs the whole-batch integration after this increment closes.

## Acceptance criteria

- [x] A hermetic e2e test (temp dirs, throwaway git repos, never the real
      workspace/`~/.axiom`) chains: adopt a legacy sandbox → `<project>.axiom`
      created + legacy byte-unchanged + migration manifest + provenance →
      read a migrated artifact (+ topology + migration manifest + adoption
      state) via the unified `axiom` MCP broker → start a worktree-mode
      execution (worktreeAdd + Execution + provision) → close it (harvest +
      cleanup) → `axiom eject` dry-run lists only rollback-eligible → RTK +
      concision skills present in the catalog with valid bundleHashes →
      autoskills lock carries `provenance: autoskills`.
- [x] Where a full hop was impractical hermetically, the SEAM between two
      increments is covered instead, and what was stubbed is stated
      explicitly in the test file itself (no silent gaps).
- [x] An adversarial review reads the shipped code (not just the specs)
      across the batch, hunting for real defects — with file:line, the
      concrete failing scenario, severity, and a recommended fix per finding.
- [x] The known W6 deferral (write-scope review / functional-verify
      validating `projectRoot`, not the worktree, on `complete`) is assessed
      for real impact — not just cited.
- [x] Provider-set/doctor consistency after the cmm swap, bundleHash
      consistency across the catalog, and adoption writer/reader convergence
      are explicitly checked (not assumed).
- [x] `npm run build` (tsc -b) passes.
- [x] The new e2e file passes; a full `npx vitest run` confirms the whole
      batch (all 16 prior increments + this one) is still green together.
- [x] Only trivial, clearly-correct defects are fixed directly; anything
      larger is reported for the orchestrator to triage.

## Open questions

None blocking.

## Assumptions

1. **The e2e chain uses the LOWER-LEVEL worktree primitives
   (`createExecutionForWorktree` + `provisionWorktreeExecution` +
   `harvestAndCleanupExecution`) for the main "happy path" chain (TEST 1),
   rather than routing through `axiom-role start/complete`.** The adopted
   `<project>.axiom` repo never runs `axiom configure` (that is a genuinely
   separate flow from `workspace adopt`), so it has no
   `install-profile.json`; composing at the primitive level keeps the main
   chain focused on the batch's OWN explicit ask (worktreeAdd + Execution +
   provision + harvest/cleanup) without dragging in `axiom-role`'s
   plan-approval/repo-affinity machinery, which is orthogonal to this
   chain's point. The higher-level `axiom-role start --worktree`/`complete`
   flow (INC-W6's actual user-facing surface) is instead exercised
   DELIBERATELY and IN DEPTH by TEST 2 and TEST 3, precisely because that is
   where the review's centerpiece findings live.
2. **TEST 1 explicitly asserts (not silently tolerates) that
   `provisionWorktreeExecution`'s adapter-surface/MCP-config steps degrade to
   warnings** (since the adopted repo has no install profile) — this is the
   one hop this increment's brief allows to be "stubbed," and it is called
   out both in the test's own comments and in this spec, not hidden.
3. **TEST 2/TEST 3 build a SEPARATE, fully `axiom init`+`axiom configure`d
   repo** specifically because reproducing the review's two centerpiece
   findings (dirty-worktree hard-stop; review/verify gates checking the
   wrong directory) requires a REAL native MCP config and a REAL
   `axiom-role start --worktree` → `complete` cycle — the adopted axiom repo
   from TEST 1 is not the right fixture for that (no code-intel/adapter
   surface is ever provisioned there, by design, so the dirty-worktree seam
   would never trigger).
4. **No product code was changed.** Every finding below was left for the
   batch orchestrator to triage, per this increment's own brief ("Fix ONLY
   trivial, clearly-correct defects... For anything larger, REPORT it").

## Implementation notes

New file: `apps/cli/tests/e2e/cross-cutting-batch.e2e.test.ts` (3 tests):

- **TEST 1** — the cross-cutting chain: adopt a legacy sandbox (INC-A2) →
  `<project>.axiom` created + legacy byte-unchanged + provenance/manifest
  (INC-A3) → `git init` the newly-adopted repo (hermetic; adoption itself
  never runs `git init`) → read the migrated increment + topology + THE REAL
  migration manifest + adoption state through the REAL unified `axiom` MCP
  broker (INC-A5, `createMcpServer({serverKind:'axiom'})`) — this is the
  FIRST test anywhere in the repo to drive the REAL migration-manifest
  WRITER's output through the REAL `axiom.migrationManifestRead`/
  `axiom.adoptionStateRead` READER (every prior test on either side used a
  hand-authored fixture matching only its own side's assumptions — see
  Finding 3) → `createExecutionForWorktree` + `provisionWorktreeExecution`
  (INC-W1+W2+W3, explicitly asserting the adapter/MCP-config stub) →
  `harvestAndCleanupExecution` (INC-W5) → `axiom eject --dry-run` (INC-A4) →
  RTK + concision skills (INC-T2/T3) verified in Axiom's OWN real catalog
  with independently-recomputed bundleHashes → AutoSkills (INC-S1)
  `provenance: autoskills` stamped in a separate hermetic code-repo sandbox.
- **TEST 2** — reproduces Finding 1 (below) end-to-end: a fully `axiom
  configure`d repo (committed native MCP config) → `axiom-role start
  --worktree` (provisioning rewrites that SAME file to the worktree's own
  path) → `axiom-role complete` (no flags) → the close HARD-STOPS on
  `dirty-worktree` even though nothing real was left uncommitted; the SDD
  role state is already persisted `archived`; `closeWorktreeExecution(...,
  {force:true})` (not CLI-exposed) is the only recovery.
- **TEST 3** — reproduces Finding 2 (below) end-to-end: a worktree-mode
  role's `complete` runs its REAL (non-bypassed) functional-verify gate; the
  worktree's own (uncommitted) `package.json` has a deliberately FAILING test
  script while the source repo's committed one PASSES; `complete` reports
  the gate as `status: 'pass'`, `repoRoot: <source repo>` — never touching
  the worktree.

## Validation

Ran from `C:\repos\Axiom Workspace\Axiom`:

- `npm run build` (`tsc -b`) — PASSED, clean, exit 0 (run multiple times
  across the iteration).
- Targeted `npx vitest run apps/cli/tests/e2e/cross-cutting-batch.e2e.test.ts`
  — **3/3 tests passed** (stable across repeated runs).
- Full-suite `npx vitest run` — **317 files, 3205 tests passed, ZERO
  failures** (up from INC-W7's own last-recorded baseline; the growth
  reflects every test file the batch's own increments added, W2 through W7,
  plus this increment's 1 new file / 3 new tests). No flaky-under-load
  timeouts reproduced this run (`context.test.ts`/`workspace-setup.test.ts`
  both green).

## Result

Implemented. The new e2e file chains the batch as specified and additionally
reproduces (not just cites) two genuine, high-severity cross-increment
defects, plus surfaces one structural fragility and two lower-confidence
observations. See the executor's final report (returned to the orchestrator)
for the full ranked findings list with file:line references, concrete
scenarios, severities, and recommended fixes. Summary of the CONFIRMED,
high-severity findings:

1. Worktree-mode `axiom-role complete` cannot cleanly close a REAL
   provisioned worktree (provisioning's own rewrite of the native MCP config
   makes the worktree "dirty" from git's perspective); no CLI-exposed force
   lever exists; the SDD role state is already persisted `archived` before
   the failure surfaces, leaving an orphaned worktree and a confusing
   exit-code/state mismatch.
2. Functional-verify (and, by identical construction, write-scope review) on
   `complete` validates `projectRoot`, never `execution.worktreePath` — a
   worktree-mode role's `complete` can report both gates PASSING even when
   the agent's actual (uncommitted) work in the worktree is broken or
   out-of-scope. This is the real-world impact of the deferral INC-W6's own
   spec already documented (Assumption #5); this review confirms the loss of
   protection is total (100%), not partial.

Both were reproduced empirically (not just deduced from reading the code) by
the new e2e test's TEST 2 and TEST 3. No product code was changed — every
finding is left for the batch orchestrator to triage.

## General spec integration

Realizada al cierre del batch INC-20260724-* (este incremento no aporta
comportamiento de producto estable propio — es un barrido e2e + revisión
adversarial):

- `04_Flujos_SDD_y_Ciclo_de_Vida.md` — mención breve del barrido e2e
  transversal y de los 2 defectos HIGH de cierre en worktree que encontró y
  reprodujo (arreglados en `INC-20260724-worktree-close-correctness`).
- Ningún otro fichero `00–08` recibe conocimiento propio: no cambió código de
  producto; su valor (los 2 hallazgos) se materializó vía el fix W8, cuya
  integración se detalla en su propio README.
