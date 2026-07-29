# Increment: Adoption config scaffolding (PC-001/PC-002/TC-011/GC-001/GC-002)

Status: closed
Date: 2026-07-27

## Goal

Make `runWorkspaceSetup` (the shared engine behind BOTH `axiom workspace
adopt` and non-adopt `axiom workspace setup`) scaffold the 4 config
artifacts that a freshly adopted/set-up project was missing, so it PASSES
5 doctor checks that a fresh project failed out-of-the-box:

- **PC-001** — `axiom.config/integrations.yaml` missing.
- **PC-002** — `axiom.config/policy-as-code.yaml` missing.
- **TC-011** — `axiom.config/agents-catalog.yaml` missing.
- **GC-001 / GC-002** — root `axiom.skills.lock` missing (the check is
  ARMED because `.axiom/{skills,agents,commands}` managed content WAS
  already being materialized by `materializeProcessSurfaces`, NS-2).

## Context

All 4 files existed in the `Axiom` product repo, but only because a
previous increment hand-committed them there — no scaffolding command ever
produced them for a newly adopted/set-up project. Two of the checks
(TC-011, GC-001/GC-002/GC-007) additionally re-compute a sha256
`bundleHash` against the materialized file content, so a naive
copy-the-product-repo's-file approach would have failed the hash
recompute (the target project's own `.axiom/agents/*`/`.axiom/skills/*`
files are NOT byte-identical to the product repo's `axiom.spec/
target-axiom-agents/*`/`target-axiom-skills/*` sources).

Precedent already existed for the "best-effort, no-clobber, committed
config file" pattern: `scaffoldArchitectDeclarations`
(`workspace-config-scaffold.ts`) seeds `mcp-manifest.yaml`/
`toolchain-catalog.yaml` the same way. And `skills-catalog.yaml` (TC-010)
already passed doctor because `scaffoldSddSkills`/`scaffoldCodeRepoSkills`
(`workspace-skills.ts`) compute `bundleHash` via `computeSkillBundleHash`
(`@axiom/skills`) against bundled seed sources it writes itself. This
increment reuses both precedents rather than inventing new ones.

## Scope

- `apps/cli/src/commands/workspace-config-scaffold.ts`: new
  `scaffoldIntegrationsYamlIfMissing`/`scaffoldPolicyAsCodeYamlIfMissing`
  (+ combined `scaffoldPolicyDeclarations`), mirroring
  `scaffoldMcpManifestIfMissing`/`scaffoldToolchainCatalogIfMissing`
  exactly (no-clobber, best-effort, atomic write). PC-001/PC-002 only
  check file EXISTENCE (no hash), so the seed content has no per-project
  data.
- `apps/cli/src/commands/workspace-process-surfaces.ts`: new exported
  `processSurfaceCatalogMeta(id)` — a thin lookup into the existing
  private `SURFACES` registry (name/responsibility/role/relatedCommand),
  so the new catalog scaffolder can describe an agent entry without
  duplicating that registry.
- `apps/cli/src/commands/workspace-catalog-scaffold.ts` (new file):
  - `scaffoldAgentsCatalogIfMissing` — scans
    `<controlRepoPath>/.axiom/agents/*.md` (the files
    `materializeProcessSurfaces` ALREADY materialized in the control
    repo), builds one `agents-catalog.yaml` entry per file with
    `source` = that file's own repo-relative path and `bundleHash` =
    `computeAgentBundleHash` (`@axiom/agents`) of that file's own
    content. No-op (returns `null`) if no `.axiom/agents/*.md` exists
    yet — nothing to catalog.
  - `scaffoldAxiomSkillsLockIfMissing` — same idea for
    `<controlRepoPath>/.axiom/skills/<id>/SKILL.md`, using
    `computeSkillBundleHash` (`@axiom/skills`). ALWAYS writes a valid
    lockfile (`installed: []` if nothing materialized yet), because
    GC-001/GC-002 only require the file to exist/be valid once
    `.axiom/*` has ANY managed content (agents/commands/skills) — not
    specifically skills.
  - Combined `scaffoldProcessSurfaceCatalogs(controlRepoPath)` wrapper
    (no-clobber per file, best-effort, never throws).
- `apps/cli/src/commands/workspace-setup.ts`: wires both new scaffolders
  into `runWorkspaceSetup` —
  `scaffoldPolicyDeclarations(control.path)` alongside the existing
  `scaffoldArchitectDeclarations(control.path)` call (step 2c-bis), and
  `scaffoldProcessSurfaceCatalogs(control.path)` right AFTER the
  `materializeProcessSurfaces` loop (step 7b-ter — order matters, the
  catalog/lockfile scanners need the `.axiom/agents/*`/`.axiom/skills/*`
  files to already exist on disk to hash them).
- New tests: `apps/cli/tests/workspace-catalog-scaffold.test.ts` (9 unit
  tests), additions to `apps/cli/tests/workspace-config-scaffold.test.ts`
  (+7 tests for PC-001/PC-002), and a new end-to-end test
  `apps/cli/tests/inc-20260727-adoption-config-scaffolding.test.ts` (3
  tests) that runs the REAL `runWorkspaceSetup` into a tmp dir and then
  runs the REAL doctor checks (`runPolicyChecks`,
  `runAgentsCatalogCoverageCheck`, `runGovernanceChecks`) against the
  result.

## Non-goals

- Wiring the new scaffolders into `member-install.ts` or
  `workspace-incremental.ts` (both also call
  `materializeProcessSurfaces` for other flows) — the brief scoped this
  explicitly to `runWorkspaceSetup` (shared by adopt + non-adopt). A
  freshly joined member's repo already has these files from the
  architect's initial `runWorkspaceSetup` run; a future increment can
  extend coverage to `member install`/`repo add` if a gap is found there.
- Changing `scaffoldArchitectDeclarations`'s existing contract/tests — a
  SEPARATE function (`scaffoldPolicyDeclarations`) was added instead, to
  avoid touching an unrelated, already-tested function's `filesCreated`
  count.
- Populating `agents-catalog.yaml`/`axiom.skills.lock` with the full
  product-repo roster (e.g. `axiom-explorer`/`axiom-reviewer`/etc., the
  `DELEGATION_ROSTER`) — those agents are never materialized as
  `.axiom/agents/*.md` in a target project (only the product repo
  itself has `axiom.spec/target-axiom-agents/*` sources for them), so
  cataloging them would immediately fail their own TC-011 hash/existence
  check. Only what `materializeProcessSurfaces` actually writes gets
  cataloged.
- Editing `Axiom.Spec/specs/00..08` (explicit instruction for this
  increment — see "General spec integration").

## Acceptance criteria

- [x] `runWorkspaceSetup` scaffolds `axiom.config/integrations.yaml` in
      the control repo (best-effort, no-clobber).
- [x] `runWorkspaceSetup` scaffolds `axiom.config/policy-as-code.yaml` in
      the control repo (best-effort, no-clobber).
- [x] `runWorkspaceSetup` scaffolds `axiom.config/agents-catalog.yaml` in
      the control repo, with one entry per `.axiom/agents/<id>.md`
      actually materialized, `bundleHash` matching TC-011's own
      recompute against that same file.
- [x] `runWorkspaceSetup` scaffolds `axiom.skills.lock` at the control
      repo's root, `schemaVersion: 1`, with one entry per
      `.axiom/skills/<id>/SKILL.md` actually materialized, `bundleHash`
      matching GC-007's own recompute against that same file.
- [x] End-to-end: a project scaffolded via `runWorkspaceSetup` into a
      tmp dir passes PC-001, PC-002, TC-011, GC-001, and GC-002 when the
      REAL doctor check functions run against it (not a
      re-implementation) — proven by
      `inc-20260727-adoption-config-scaffolding.test.ts`.
- [x] `npm run build` passes.
- [x] All new + pre-existing tests pass, including the full
      `apps/cli`/`packages/doctor`/`packages/agents`/`packages/skills`
      suites (no regression).

## Open questions

None blocking. See Assumptions.

## Assumptions

1. **The doctor checks run against the CONTROL (SDD) repo, not the spec
   or role repos.** Confirmed by reading the existing code:
   `axiom.config/` (topology.yaml, profiles.yaml, mcp-manifest.yaml,
   toolchain-catalog.yaml, skills-catalog.yaml) is ALWAYS written to
   `control.path` only, never to `spec.repos` generically. The new
   scaffolders follow the same convention.
2. **`agents-catalog.yaml`/`axiom.skills.lock` are derived from
   `.axiom/agents/*`/`.axiom/skills/*` (the PORTABLE, adapter-agnostic
   forms `materializeProcessSurfaces` always writes), not from
   `.opencode/agents/*` (adapter-specific, opt-in) or from
   `@axiom/agents`'s own `materializeAgentSet`/`loadAgentsCatalog`
   (confirmed UNUSED anywhere in `apps/cli` today — no other code path
   depends on `agents-catalog.yaml` having any particular roster).** This
   is why the control repo — whose `repoRole` is always `'sdd'` — ends up
   with exactly 3 agents entries (`axiom-sdd-orchestrator`,
   `axiom-phase-reviewer`, `axiom-qa-validator`) and 3 skills entries
   (same 3 ids) after a fresh `runWorkspaceSetup`, matching
   `surfaceIdsForRole('sdd')`.
3. **`axiom.skills.lock` entries use `source: 'axiom-process-surfaces'`**
   (a provenance TAG, not a file path — same convention as
   `reconcileOpencodeLock`'s `source: 'product-registry'` for
   `.opencode/skills-lock.yaml`), deliberately DIFFERENT from
   `'product-registry'` so GC-013 (which only validates entries with
   `source === 'product-registry'` against
   `axiom.spec/product.manifest.yaml`) never considers these entries.
4. **`agents-catalog.yaml` is NOT scaffolded (stays absent) if no
   `.axiom/agents/*.md` exists yet** — TC-011 explicitly FAILS on an
   empty `agents: []` array, so writing an empty catalog would be worse
   than not writing one at all (both are `fail`/`absent` from TC-011's
   perspective before this increment; only a POPULATED, hash-correct
   catalog is a `pass`). In every real `runWorkspaceSetup` run the
   control repo always gets its 3 process surfaces, so this path is
   theoretical (a wiring-order bug would be the only way to hit it).
   `axiom.skills.lock`, by contrast, ALWAYS writes (even `installed: []`)
   because GC-001/GC-002 only require existence/validity once `.axiom/*`
   has ANY managed content — not specifically skills.
5. **Wiring order matters and is enforced structurally**: the new
   `scaffoldProcessSurfaceCatalogs(control.path)` call sits in
   `runWorkspaceSetup` immediately AFTER the `materializeProcessSurfaces`
   loop (not before, not merged into step 2c/2c-bis) — the catalog/
   lockfile scanners read files from disk, so they must run after those
   files exist.

## Implementation notes

### Hash computation — how it's guaranteed to match the doctor's recompute

Both `scaffoldAgentsCatalogIfMissing` and `scaffoldAxiomSkillsLockIfMissing`
hash the EXACT SAME file they reference as `source`/`installedTo`:

- `agents-catalog.yaml`: for each `.axiom/agents/<id>.md` file, `source` is
  set to that file's own repo-relative path (`.axiom/agents/<id>.md`,
  posix-normalized), and `bundleHash` is
  `computeAgentBundleHash(fs.readFileSync(thatSameFile, 'utf-8'))`
  (`@axiom/agents`, `sha256:<hex>`). TC-011
  (`runAgentsCatalogCoverageCheck`, `packages/doctor/src/checks.ts`)
  resolves `entry.source` against `resolution.rootPath` and recomputes
  `sha256:` + `crypto.createHash('sha256').update(sourceContent).digest('hex')`
  — byte-identical algorithm, same file → guaranteed match.
- `axiom.skills.lock`: for each `.axiom/skills/<id>/SKILL.md` file,
  `installedTo` is that file's own repo-relative path, and `bundleHash` is
  `computeSkillBundleHash(fs.readFileSync(thatSameFile, 'utf-8'))`
  (`@axiom/skills`, same `sha256:<hex>` format). GC-007 (`checkGC007`,
  `packages/doctor/src/governance-checks.ts`) resolves `entry.installedTo`
  against `rootPath` and recomputes the identical
  `'sha256:' + crypto.createHash('sha256').update(content, 'utf-8').digest('hex')`
  — same algorithm, same file → guaranteed match.

Both scaffolders read the file ONCE and hash that exact in-memory string —
there is no re-read/re-render step that could introduce a mismatch (e.g. no
YAML round-trip of the markdown content).

### Where things are wired

- `scaffoldPolicyDeclarations(control.path)` — `workspace-setup.ts` step
  2c-bis, right after the pre-existing `scaffoldArchitectDeclarations`
  call.
- `scaffoldProcessSurfaceCatalogs(control.path)` — `workspace-setup.ts`
  step 7b-ter, right after the `materializeProcessSurfaces` loop (step
  7b-bis) and before the code-intel init step (7c).

## Validation

Ran from `C:\repos\Axiom Workspace\Axiom`:

- `npm run build` (`tsc -b`) — PASSED, clean, exit 0 (ran twice: right
  after implementation, and again after adding all new tests).
- `npx vitest run apps/cli/tests/workspace-setup.test.ts
  apps/cli/tests/workspace-config-scaffold.test.ts
  apps/cli/tests/workspace-process-surfaces.test.ts
  apps/cli/tests/workspace-adopt.test.ts
  apps/cli/tests/workspace-skills.test.ts` — **5 files / 80 tests
  passed** (confirms no regression in the exact areas touched, including
  the strict `expect(second.warnings).toHaveLength(1)` idempotent-re-run
  assertion in `workspace-setup.test.ts`).
- `npx vitest run packages/doctor/tests` — **19 files / 199 tests
  passed** (no regression in any doctor check, including
  `governance.test.ts`'s 32 tests and `agents.test.ts`'s 9).
- `npx vitest run apps/cli/tests/workspace-catalog-scaffold.test.ts` —
  **9/9 new unit tests passed** (agents-catalog: absent-when-nothing-
  materialized, hash-matches-TC-011-recompute, no-clobber;
  axiom.skills.lock: always-valid-even-empty, hash-matches-GC-007-
  recompute, no-clobber; combined wrapper: both-written,
  idempotent-second-run, best-effort-never-throws).
- `npx vitest run apps/cli/tests/inc-20260727-adoption-config-scaffolding.test.ts`
  — **3/3 new END-TO-END tests passed**: (1) all 4 files exist +
  appear in `filesCreated` after a real `runWorkspaceSetup`; (2) PC-001,
  PC-002, TC-011, GC-001, GC-002 (and, as a bonus, GC-007) all report
  `status: 'pass'` when the REAL doctor check functions
  (`runPolicyChecks`/`runAgentsCatalogCoverageCheck`/
  `runGovernanceChecks`, imported from `@axiom/doctor`, not
  re-implemented) run against `resolveProject(controlPath)`; (3) a
  second `runWorkspaceSetup` run is idempotent (no re-added files, no new
  warnings mentioning any of the 4 files).
- `npx vitest run apps/cli/tests packages/doctor/tests packages/agents/tests
  packages/skills/tests` (full regression pass across every touched/
  adjacent package) — **156 files / 1529 tests passed**, zero failures.

No pre-existing failures were encountered in any touched or adjacent scope.

## Result

Implemented. `runWorkspaceSetup` now scaffolds all 4 previously-missing
config artifacts in the control repo, best-effort and no-clobber:
`axiom.config/integrations.yaml` (PC-001) and `axiom.config/
policy-as-code.yaml` (PC-002) via the new `scaffoldPolicyDeclarations`
(mirroring `scaffoldArchitectDeclarations`); `axiom.config/
agents-catalog.yaml` (TC-011) and root `axiom.skills.lock` (GC-001/
GC-002/GC-007) via the new `scaffoldProcessSurfaceCatalogs`, which scans
the `.axiom/agents/*`/`.axiom/skills/*` files `materializeProcessSurfaces`
already writes and hashes them with the SAME helpers
(`computeAgentBundleHash`/`computeSkillBundleHash`) the doctor's own
recompute uses — guaranteeing the hash always matches. Proven end-to-end:
a project scaffolded from scratch via `runWorkspaceSetup` now passes all
5 previously-failing checks when the real doctor check functions run
against it. Full regression (`apps/cli`, `packages/doctor`,
`packages/agents`, `packages/skills` — 156 files / 1529 tests) is green.

## General spec integration

Integrated into `Axiom.Spec/specs/00_Resumen_Ejecutivo.md`,
`Axiom.Spec/specs/03_Modelo_Operativo_y_Datos.md`, and
`Axiom.Spec/specs/06_Integraciones_y_Capacidades.md`: the shared setup
engine now scaffolds the four artifacts and preserves the doctor/hash
contracts described above. Technical-context outcome: the same current
behavior is recorded in
`Axiom.Spec/context/architecture/02-modelo-de-datos-y-configuracion.md`,
`Axiom.Spec/context/operations/01-instalacion-y-onboarding.md`, and
`Axiom.Spec/context/TECHNICAL_CONTEXT.md`.
