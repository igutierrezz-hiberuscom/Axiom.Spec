# Increment: lifecycle correctness fixes (P2 correctness + minor bugs)

Status: closed
Date: 2026-07-10

## Goal

Fix five independent, audit-confirmed correctness bugs across the
increment/bug lifecycle, `self-update`, `upgrade`, `bootstrap
from-legacy-sdd`, and registry v1→v2 migration — each small, minimal,
and well-tested, with zero regressions to the existing green suite.

## Context

Five unrelated P2 bugs were audit-confirmed in `Axiom/`:

1. `axiom-increment archive` / `axiom-bug archive` only flipped
   `status` to `'archived'` in `metadata.yml`; they never physically
   moved the artifact folder into an `_archive/` sibling, contradicting
   the established convention (the spec repo itself already uses
   `specs/increments/_archive/` for archived increments).
2. `readInstalledVersion()` (`@axiom/user-workspace`) read
   `<process.cwd()>/package.json` — comparing against the OPERATOR's
   project directory, not the installed CLI package — so `self-update
   --check/--apply` compared against the wrong version depending on
   where the command was invoked from.
3. `axiom upgrade`'s CLI wrapper prepended `'gateFailure: '` to an
   `Error.message` that already began with `'gateFailure: true — …'`
   (the message format used by `GateFailureError`,
   `@axiom/orchestrator`'s `runner.ts`), producing the doubled prefix
   `'gateFailure: gateFailure: true — …'` in stderr.
4. `axiom bootstrap from-legacy-sdd --dry-run`'s preview used
   `resolveWorkflowStateForMigration(rawStatus)` for ALL five artifact
   kinds, but the REAL migration for `adr`/`decision` always hardcodes
   `status: 'proposed'` (INC-11 design — not state-machine-driven). An
   `adr` with legacy `rawStatus: 'accepted'` previewed `status: draft`
   but the real migration produced `status: proposed` — a lying
   preview.
5. `axiom init` and `axiom repo attach` refused to register a project
   when only a legacy `~/.axiom/registry.json` (v1) existed (no
   `projects.yml` v2 yet), telling the operator to run `axiom upgrade`
   — a command that does not support this migration. The wizard path
   (`runWorkspaceSetup`) already auto-migrated via
   `migrateLegacyRegistryV1ToV2` for the exact same cause
   (`INC-20260705-workspace-setup-registry-robustness`); `init`/`repo
   attach` never adopted that fix.

## Scope

- `packages/workflow/src/artifact-store.ts`: new `archiveArtifactDir` +
  `resolveArchivedArtifactDir`, exported from `@axiom/workflow`'s
  barrel. Physically moves (atomic rename) `<kindFolder>/<id>/` to
  `<kindFolder>/_archive/<id>/`; never clobbers an existing archived
  folder.
- `apps/cli/src/commands/axiom-increment.ts` /
  `apps/cli/src/commands/axiom-bug.ts`: `syncIncrementMetadata` /
  `syncBugMetadata` call `archiveArtifactDir` right after persisting
  `status: 'archived'`, when `toState === 'archived'`. Best-effort
  (stderr warning on failure; never blocks the already-computed state
  transition).
- `packages/user-workspace/src/self-update.ts`: `readInstalledVersion`
  rewritten to walk up from `__dirname` (module location) to the
  nearest `package.json`, instead of reading
  `<process.cwd()>/package.json`.
- `apps/cli/src/commands/upgrade.ts`: extracted
  `formatUpgradeErrorLine` (pure, testable) from the `registerUpgrade`
  action's catch block; fixed the `gateFailure` branch to not
  re-prepend the already-present prefix.
- `apps/cli/src/bootstrap-from-legacy-sdd/migrator.ts`: new
  `resolveDisplayStatusForMigration(kind, rawStatus)` — single source
  of truth for the status shown by BOTH the `--dry-run` preview and
  the real migration (adr/decision always `'proposed'`; increment/bug/
  plan delegate to the existing `resolveWorkflowStateForMigration`).
- `apps/cli/src/commands/bootstrap.ts`: `--dry-run` preview now uses
  `resolveDisplayStatusForMigration` instead of
  `resolveWorkflowStateForMigration` directly.
- `apps/cli/src/commands/init.ts` / `apps/cli/src/commands/repo.ts`:
  on `legacy-registry-not-migrated`, both now call
  `migrateLegacyRegistryV1ToV2` once and retry the lookup/upsert once
  (same pattern as `runWorkspaceSetup`), instead of refusing with a
  dead-end `axiom upgrade` pointer.
- `Axiom.Spec/specs/03_Modelo_Operativo_y_Datos.md` and
  `Axiom.Spec/specs/04_Flujos_SDD_y_Ciclo_de_Vida.md`: canonical
  knowledge updates for fixes #1 and #5 (see "General spec
  integration" below).

## Non-goals

- Did NOT fix the analogous `upgradeFailed:` double-prefix in the same
  `upgrade.ts` catch block (`upgradeFailed: upgradeFailed: true — …`).
  It shares the exact same root cause as the `gateFailure` bug but was
  not named in the audit-confirmed scope for this increment; flagged
  below as a discovered, out-of-scope sibling bug for a fast follow.
- Did NOT expand `listArtifacts`/`axiom-increment list` to also scan
  `<kindFolder>/_archive/*`. After fix #1, an archived artifact no
  longer appears in the default listing (documented explicitly as
  expected in `04_Flujos_SDD_y_Ciclo_de_Vida.md`). Making `list` archive-
  aware is a reasonable follow-up but is new behavior, not a bug fix,
  and was not requested.
- Did NOT change `readInstalledVersion`'s signature (still zero-arg) —
  kept the fix as a pure internal resolution-strategy change to
  minimize blast radius.
- No changes to `axiom-adr`/`axiom-decision`/`axiom-plan` archive
  behavior (none of the three have an `archive` transition to
  `'archived'` today, so fix #1's scope is exactly increment + bug, per
  the audit).

## Acceptance criteria

- [x] `archive` (increment and bug) physically relocates the artifact
      folder to `<kindFolder>/_archive/<id>/`; metadata still resolves
      (readable on disk) at the new location; a pre-existing
      destination folder causes a clear `err`, never a clobber.
- [x] `readInstalledVersion()` no longer depends on `process.cwd()`.
- [x] `axiom upgrade`'s `gateFailure` stderr line contains the prefix
      exactly once, never doubled.
- [x] `axiom bootstrap from-legacy-sdd --dry-run`'s preview status for
      an `adr`/`decision` item matches EXACTLY what the real migration
      produces (`'proposed'`), for the same input.
- [x] `axiom init` and `axiom repo attach` succeed (auto-migrate) when
      only a legacy v1 registry is present, instead of refusing.
- [x] `npm run build` exits 0.
- [x] Live e2e proof (temp project, real compiled CLI binary): drove a
      real increment AND a real bug through their full create→archive
      chain; confirmed the folder physically moved under `_archive/`
      in both cases.
- [x] Targeted `vitest run` on touched files + full `npm test` green,
      zero regressions (only additions).

## Open questions

None blocking. One discovered-but-unfixed sibling bug is documented
under "Non-goals" (the `upgradeFailed:` double-prefix) — recommend a
tiny fast-follow bug ticket if desired; not fixed here per the
increment's literal audit-confirmed scope (only `gateFailure:` was
named).

## Assumptions

- "Fail with a clear message rather than clobber" (fix #1) was
  interpreted at the `archiveArtifactDir` primitive's contract level
  (returns `err`, never overwrites), not as a requirement to change
  the increment/bug `archive` command's overall exit code — the
  existing `syncIncrementMetadata`/`syncBugMetadata` design is already
  documented as best-effort or metadata-sync side effects (state
  transition success is independent of metadata-sync success); the
  archive-move failure path follows that same, already-established
  contract (stderr warning, no exit-code change).
- Fix #2's "installed package directory" is resolved via the standard
  "nearest `package.json` walking up from the running module's
  `__dirname`" heuristic (mirroring the existing `findInstallScript`
  pattern in `apps/cli/src/commands/self-update.ts`), not via
  `require.main`/`import.meta.url` — sufficient for both the dev
  monorepo and a distributed install, per the increment's own wording.
- Fix #5's error message wording (for the case where the migration
  itself fails) was written fresh for `init`/`repo attach` rather than
  verbatim-copied from `runWorkspaceSetup`'s warning, since the exit
  semantics differ (`init` degrades to a WARN + `registryRegistered:
  false`; `repo attach` exits 1) — same information content, phrasing
  adapted to each command's existing message contract.

## Implementation notes

Files changed (with 1-line rationale each):

1. **Archive physical move**
   - `packages/workflow/src/artifact-store.ts` — added
     `archiveArtifactDir`/`resolveArchivedArtifactDir` (atomic rename
     to `_archive/`, never-clobber contract).
   - `packages/workflow/src/index.ts` — barrel-exported both new
     functions.
   - `apps/cli/src/commands/axiom-increment.ts` — `syncIncrementMetadata`
     calls `archiveArtifactDir` after `toState === 'archived'`.
   - `apps/cli/src/commands/axiom-bug.ts` — same, `syncBugMetadata`.
   - Tests: `packages/workflow/tests/artifact-store.test.ts` (+3: move
     success, not-found source, no-clobber collision).

2. **`readInstalledVersion` cwd independence**
   - `packages/user-workspace/src/self-update.ts` — walk-up from
     `__dirname` to nearest `package.json`, replacing the
     `process.cwd()` read; updated AD-05/JSDoc.
   - `apps/cli/src/commands/self-update.ts` — updated header comment
     (no code change; it only calls the function).
   - Tests: `packages/user-workspace/tests/self-update.test.ts` (+1:
     mocked `process.cwd` to a decoy dir with a different
     `package.json`, asserted the result is unaffected).

3. **`gateFailure:` double prefix**
   - `apps/cli/src/commands/upgrade.ts` — extracted
     `formatUpgradeErrorLine` (pure function); fixed the `gateFailure`
     branch to not re-prepend the prefix already present in `message`.
   - Tests: new `apps/cli/tests/upgrade.test.ts` (4 tests: gateFailure
     single-prefix, upgradeFailed unchanged-behavior-documented,
     generic error, non-Error throw).

4. **Dry-run preview vs real migration status mismatch (adr/decision)**
   - `apps/cli/src/bootstrap-from-legacy-sdd/migrator.ts` — added
     `resolveDisplayStatusForMigration(kind, rawStatus)`, the single
     source of truth for both preview and real migration.
   - `apps/cli/src/commands/bootstrap.ts` — `--dry-run` preview now
     calls the new function instead of
     `resolveWorkflowStateForMigration` directly.
   - Tests: `apps/cli/tests/bootstrap-from-legacy-sdd/migrator.test.ts`
     (+4) and `apps/cli/tests/bootstrap.test.ts` (+1 integration test:
     preview vs real for an `accepted` ADR).

5. **Registry v1→v2 auto-migration in `init`/`repo attach`**
   - `apps/cli/src/commands/init.ts` — on
     `legacy-registry-not-migrated`, calls `migrateLegacyRegistryV1ToV2`
     once, retries the `findByRepoPathV2` lookup once.
   - `apps/cli/src/commands/repo.ts` — same pattern for `addProjectV2`
     in `runRepoAttach`.
   - Tests: `apps/cli/tests/init.test.ts` (+1) and
     `apps/cli/tests/repo.test.ts` (+1).

## Validation

- `cd Axiom && npm run build` → exit 0 (`tsc -b`, clean, no errors),
  run twice (immediately after all 5 fixes, and again after all test
  additions).
- Targeted `npx vitest run` on touched files (workflow, user-workspace,
  and the apps/cli files touched): **all green**
  (`packages/workflow/tests/artifact-store.test.ts`: 35 tests;
  `packages/user-workspace/tests/self-update.test.ts`: 14 tests;
  `packages/user-workspace/tests/migrate-legacy-registry.test.ts`: 5
  tests; `apps/cli/tests/axiom-increment.test.ts`: 10;
  `apps/cli/tests/axiom-increment-metadata.test.ts`: 4;
  `apps/cli/tests/axiom-bug.test.ts`: 6;
  `apps/cli/tests/axiom-bug-metadata.test.ts`: 3;
  `apps/cli/tests/upgrade.test.ts`: 4 (new file);
  `apps/cli/tests/bootstrap.test.ts`: 17;
  `apps/cli/tests/bootstrap-from-legacy-sdd/migrator.test.ts`: 17;
  `apps/cli/tests/init.test.ts`: 17; `apps/cli/tests/repo.test.ts`: 7).
- LIVE e2e proof for fix #1, in a session-scratchpad temp project
  (never touching the real repo or a real user home), using the REAL
  compiled `apps/cli/dist/index.js` binary (no mocks):
  - Increment: `create --id 7777 --slug live-archive-test` →
    `specify` → `plan-approve` → `verify` → `archive`, all exit 0.
    Confirmed on disk: `axiom.spec/increments/` no longer has a
    top-level `INC-…` folder; `axiom.spec/increments/_archive/INC-
    20260710-132251-xz4tvj/{metadata.yml,README.md}` exists, with
    `metadata.yml` showing `status: archived`.
  - Bug: same chain (`create` → `fix-plan` → `verify` → `archive`),
    same result: `axiom.spec/bugs/_archive/BUG-20260710-132312-
    cproq6/{metadata.yml,README.md}` exists.
  - Scratch directory removed after the proof.
- Corrected message/preview outputs (unit tests, fix #3/#4):
  - Fix #3: `formatUpgradeErrorLine` on a `gateFailure` error whose
    message already reads `'gateFailure: true — command="upgrade"
    reason="no init.json" remediation="corré axiom init"'` now returns
    `'[axiom upgrade] gateFailure: true — command="upgrade"
    reason="no init.json" remediation="corré axiom init"'` — the
    string `'gateFailure:'` appears exactly once (test asserts
    `.split('gateFailure:').length - 1 === 1`).
  - Fix #4: for an ADR item with `rawStatus: 'accepted'`, both the
    `--dry-run` preview line and the real migration's created-artifact
    line now show `(status: proposed)` — verified equal in the same
    test (`previewStatus === realStatus === 'proposed'`), and via a
    full `runBootstrapFromLegacySdd` dry-run vs real-run comparison at
    the CLI-command level.
- Full `npm test` (full `vitest run`, all 212 files): **212 files /
  2254 tests, all pass, zero failures.** Pre-increment baseline (per
  the brief): 211 files / 2239 tests. Delta: +1 test file
  (`upgrade.test.ts`) and +15 tests total across the touched files
  (3 + 1 + 4 + 1 + 1 + 1 + ... — see per-file counts above), matching
  exactly (2239 + 15 = 2254) — confirms zero regressions, only
  additions.

## Result

Closed. All five audit-confirmed bugs were fixed with the smallest
correct change each: a new (never-clobbering, atomic) archive-move
primitive reused identically by increment and bug; a cwd-independent
version-resolution walk-up mirroring an existing pattern in the same
file family; a one-line duplicate-prefix removal backed by an extracted
pure formatter function; a single shared status-resolution function
consumed by both the preview and the real migration path; and the
exact same auto-migrate-once-and-retry-once pattern already proven by
the wizard, now reused by `init` and `repo attach`. Live e2e proof
covers fix #1 end-to-end with the real compiled binary; the remaining
four are covered by targeted unit/integration tests. The full suite
stayed green with zero regressions (2254/2254, +15 new tests, +1 new
test file). One discovered-but-out-of-scope sibling bug
(`upgradeFailed:` double prefix, same root cause as fix #3) is
documented under "Non-goals" for a possible fast follow.

## General spec integration

- `Axiom.Spec/specs/04_Flujos_SDD_y_Ciclo_de_Vida.md` — added point 6
  under "Ciclo de vida de artefactos (increment/bug/plan/ADR/decision)"
  documenting the new physical archive-move behavior (fix #1),
  including the explicit, intentional side effect that
  `listArtifacts`/`axiom-increment list` no longer surfaces an archived
  artifact by default (consistent with the spec repo's own pre-existing
  `_archive/` convention).
- `Axiom.Spec/specs/03_Modelo_Operativo_y_Datos.md` — updated the
  "Registro no bloqueante y auto-migración de registry v1→v2" section
  (originally written by `INC-20260705-workspace-setup-registry-
  robustness`), which explicitly stated "Alcance actual: solo la ruta
  de `runWorkspaceSetup`; `runInit`/`axiom init` conserva su propio
  manejo previo... sin cambios" — now OUTDATED by fix #5. Replaced with
  a note that `axiom init` and `axiom repo attach` now reuse the same
  auto-migration, so all three paths (`runWorkspaceSetup`, `runInit`,
  `runRepoAttach`) behave consistently.
- Fixes #2, #3, #4 are implementation-level correctness fixes with no
  canonical product-behavior change worth a separate doc update beyond
  what's already stated above (the observable CLI contract for
  `self-update`/`upgrade`/`bootstrap from-legacy-sdd` was already
  correctly documented; only the internal implementation was wrong).
  `general-spec.md` does not exist in this repo's structure (same
  situation noted by prior increments, e.g.
  `INC-20260710-dynamic-team-roles`,
  `INC-20260710-workspace-command-parity`); the two numbered spec docs
  above are the equivalent canonical location used here.
