# INC-20260711-repo-affinity-guard

## Goal

Add a **repo-affinity guard** so that lifecycle commands are enforced against
WHICH repo the operator (human or agent) is currently in, in a multi-repo Axiom
workspace where roles↔repos are defined. An agent that opens the wrong repo and
launches a lifecycle command is told "no", with a clear, actionable message.

Concretely:

- `axiom-increment`, `axiom-bug`, `axiom-plan` must run from the **SPEC repo**.
  Running them from the control (SDD) repo or a role/code repo is rejected.
- `axiom-role` (`start`/`apply`/`complete`/`sync-graph`) for role **X** must run
  ONLY from the repo assigned to role **X**. Opening the BACK repo and launching
  the FRONT role is rejected ("you're not in the correct repo"); from the BACK
  repo you may only launch the BACK role.

## Scope

- Surface repo identity (`role`, `repoId`) as first-class fields on
  `ProjectResolution` (additive; parsed from `axiom.yaml` schemaVersion 2).
- Materialize the role→repo map (`axiom.config/topology.yaml`) into EVERY repo
  of a multi-repo workspace (control + spec + each role repo), anchored per-repo,
  so any repo can answer "what is my role, and which repo owns role X" via
  `loadTopology(currentRepo)`. Wired into `runWorkspaceSetup`, `runRepoAdd`
  (incremental `repo add`/`role add`), and `axiom member install` (best-effort
  materialization of the shared map into a freshly cloned repo when missing).
- A shared guard helper `checkRepoAffinity(...)` (new module
  `apps/cli/src/commands/_repo-affinity.ts`) mirroring the existing
  `checkPlanIsApproved` pattern — returns `Result<undefined,{message,exitCode:1}>`.
- Wire the guard into the four lifecycle entrypoints
  (`axiom-increment`, `axiom-bug`, `axiom-plan`, `axiom-role`).

## Non-goals

- No user-facing flag that defeats the feature. (An internal `skipAffinityGuard`
  param exists on the run functions purely for programmatic/test bypass.)
- No change to the workflow state machines, artifact metadata, or the
  `checkPlanIsApproved` safety gate.
- No enforcement in single-repo / self-hosted / unresolvable projects.

## States & rules (the guard matrix)

The guard is a strict **NO-OP (allow)** unless ALL of the following hold:

1. `loadTopology(currentRepo)` succeeds AND `manifest.mode === 'multi-repo'`.
2. `resolveProject(currentRepo).status === 'resolved'` AND `resolution.role` is a
   non-empty string (i.e. the repo has a schemaVersion-2 `axiom.yaml`).
3. `manifest.assignments.length > 0`.

When enforcing:

| Command                              | Current repo role | Rule                                                                 | Result   |
|--------------------------------------|-------------------|----------------------------------------------------------------------|----------|
| `axiom-increment` / `axiom-bug` / `axiom-plan` | `spec`   | current repo is the spec repo                                        | ALLOW    |
| `axiom-increment` / `axiom-bug` / `axiom-plan` | `sdd`/`code` | wrong repo — must run from spec                                    | REJECT   |
| `axiom-role X`                       | `code`, assigned role == X | current repo owns role X                                    | ALLOW    |
| `axiom-role X`                       | `code`, assigned role != X | wrong role repo (e.g. FRONT from BACK)                      | REJECT   |
| `axiom-role X`                       | `sdd`/`spec`      | not a role/code repo                                                  | REJECT   |
| any                                  | (gate not met)    | single-repo / v1 axiom.yaml / no assignments                         | NO-OP    |

Target roleId for `axiom-role` is derived from the operated role's `--slug`/`--id`
matched against the registered team roles in `topology.yaml#roles`/`#assignments`.
When the target roleId cannot be determined, the guard still requires the current
repo to be a valid (assigned) code repo, but does not cross-check the role name.

Rejection message format (Spanish UI, English identifiers), tagged
`[axiom repo-affinity]`:

- spec: `Este comando debe ejecutarse desde el repo de SPEC del workspace. Estás en el repo '<role>' (repoId=<repoId>). El repo de spec es '<specRepoId>'[ (<absPath>)].`
- role (wrong role): `No estás en el repo correcto para el rol '<targetRoleId>'. Estás en el repo del rol '<currentRoleId>' (repoId=<repoId>). Ejecutá este comando desde el repo asignado al rol '<targetRoleId>' ('<targetRepoId>'[ (<absPath>)]).`
- role (not a role repo): `axiom-role debe ejecutarse desde un repo de código asignado a un rol de equipo. Estás en el repo '<role>' (repoId=<repoId>).`

## Acceptance criteria

- AC1: Multi-repo — `axiom-increment`/`axiom-bug`/`axiom-plan` from the SPEC repo
  are ALLOWED; from a role/control repo are REJECTED with exit code 1 and a clear
  message.
- AC2: Multi-repo — `axiom-role start` for role FRONT from the FRONT repo is
  ALLOWED; from the BACK repo is REJECTED, naming the correct repo.
- AC3: single-repo / no-topology / unresolvable repo / missing-or-empty
  assignments → guard is a NO-OP (command proceeds, no crash).
- AC4: The role→repo map (`topology.yaml`) is materialized into every repo by
  `runWorkspaceSetup` / `runRepoAdd` (and best-effort by `member install`).
- AC5: Full build green; the entire existing test suite stays green; new unit +
  integration tests cover the matrix above.

## Result

Implemented and verified end-to-end. Status `implemented`.

- Build: GREEN (`tsc -b`).
- Tests: GREEN — 214 files, 2298 tests (was 2275; +23 new in
  `apps/cli/tests/repo-affinity.test.ts`). No existing test weakened or deleted.
- e2e (built CLI over a real 3-repo `workspace setup`): increment/plan from the
  SPEC repo ALLOWED; increment/bug from a role repo REJECTED; `axiom-role start`
  FRONT from the BACK repo REJECTED (names the front repo + abs path); BACK role
  from the BACK repo (plan approved) ALLOWED. `topology.yaml` materialized into
  all four repos.

Files changed:

- `packages/project-resolution/src/resolver.ts` — additive `role`/`repoId` on
  `ProjectResolution`, populated in `resolveFromV2`.
- `apps/cli/src/commands/workspace-setup.ts` — exported `deriveRepoId`;
  `buildTopologyManifestAnchoredAt`; materialize topology into every repo.
- `apps/cli/src/commands/_repo-affinity.ts` (new) — `checkRepoAffinity` guard.
- `apps/cli/src/commands/axiom-increment.ts` / `axiom-bug.ts` / `axiom-plan.ts`
  / `axiom-role.ts` — guard wired in (+ internal `skipAffinityGuard`).
- `apps/cli/src/commands/workspace-incremental.ts` — `runRepoAdd` materializes
  topology into every repo (per-repo anchored).
- `apps/cli/src/commands/member-install.ts` — best-effort materialization of the
  shared map into the joined repo when missing.
- `apps/cli/tests/repo-affinity.test.ts` (new) — unit + integration coverage.
