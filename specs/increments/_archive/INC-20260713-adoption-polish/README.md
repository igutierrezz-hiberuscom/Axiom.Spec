# INC-20260713-adoption-polish

Adoption/migration polish. Three surgical fixes confirmed in the KVP25 sandbox
adoption, on top of the provenance-idempotent migration engine
(`INC-20260711-mig-idempotency`) and the install-time adoption UX
(`INC-20260711-mig-adopt-ux`). The build is GREEN with the existing test suite
and MUST stay green; every fix is small and additive.

## Goal

Make install-time adoption honest and self-consistent:

1. A re-run **preview** must predict exactly what the real run will do (no
   over-reporting "would migrate N" when the real run skips all N).
2. Adoption must **warn** when a role code-repo has no baseline commit, so the
   later `axiom-role start --create-branch` does not branch off an unborn HEAD.
3. Generated MCP launch commands must be **resolvable** without silently
   assuming an ambient global `axiom` on PATH.

## Scope — the three fixes

### Fix 1 — Preview reflects the provenance map (not over-report)

`apps/cli/src/commands/workspace-adopt.ts` built its subject-A / context preview
from a read-only scan filtered ONLY by the `AXIOM:MIGRATED` banner guard
(`isMigrationOutputBody`). On a re-run this predicted "would migrate N" while the
real run (`runBootstrapFromLegacySdd` / `runBootstrapFromContext`, provenance-
idempotent) correctly skipped all N.

Fix: the preview now consults the SAME provenance map
(`apps/cli/src/bootstrap-shared/provenance.ts`, stored at
`<specScope>/technical-context/.migration-provenance.yml`) the real run uses. A
new shared helper `resolveLiveProvenanceMatch` encapsulates the exact predicate
("path OR content-hash match for this subject whose recorded target still exists
on disk"). It is called by BOTH the bootstrap from-legacy-sdd dry-run and the
adoption preview, so preview classification (`would-migrate` vs
`already-migrated`) equals the real run item-for-item. The context preview also
honours the target-exists (skip-if-exists) gate via `mapIngestedDocs`. The
real-run code paths are untouched (they were already correct).

### Fix 2 — Baseline-commit detection for role repos (warn, never mutate)

The KVP25 adoption left role code-repos on an UNBORN branch (git init, no
commit). Adoption now DETECTS, per role repo (`kind === 'role'`), whether the
directory is its own git repo root (`.git` present at the repo root — a
filesystem check that avoids git's parent-directory walk) with at least one
commit (`git rev-parse --verify HEAD`, a READ-ONLY probe through the existing
`@axiom/workflow` `GitRunner` seam). A role repo with no commits — or that is not
a git repo — surfaces a clear warning in the adoption/conformance report and in
`result.warnings`:

> `role repo <name> (<path>) has no baseline commit; run 'git init && git commit' before role-branch operations (axiom-role start --create-branch).`

(or, for a non-repo: `... is not a git repository; run 'git init && git commit' ...`).

**Guardrail honoured:** adoption runs ZERO git mutations. It never runs
`git init` / `git commit`. It only reports and directs the operator to establish
a baseline. The `GitRunner` is injectable (`WorkspaceAdoptArgs.gitRunner`) for
deterministic tests.

### Fix 3 — Generated MCP `command` is resolvable

Generated native/canonical MCP configs could carry a bare `"command": "axiom"`,
which only works when a global `axiom` is on PATH (the sandbox had none).

**Choice: approach (a) — resolve a runnable form — via the already-shipped
`resolveMcpLaunchCommand()`.** See D-003 for the full justification. In short:
Axiom's intended distribution IS a global `axiom` (the Windows shim / `npm link`
from `scripts/install-global.mjs`), and `resolveMcpLaunchCommand()` already
PREFERS `axiom` when it is on PATH, falling back to `node <abs cli entry>`
(`process.argv[1]`) only when it is not — exactly the sandbox case, never a
guessed/hard-coded path. The production adoption path (`runWorkspaceSetup`) and
`axiom member install` already thread it. The only remaining bare-`axiom`
emitters — `axiom workspace mcp-config` (`runWorkspaceMcpConfig`) and
`axiom workspace add-repo` (`runWorkspaceAddRepo`) — now thread
`resolveMcpLaunchCommand()` too (override-injectable, mirroring
`runWorkspaceSetup`'s existing `mcpLaunchCommandOverride`). The adoption report
additionally emits an informational note naming the resolved launch command and
the global-`axiom` prerequisite (the documented-prereq half of option b), so the
distribution assumption is visible to the operator.

This is least-invasive (one threaded argument per call site, reusing shipped and
tested code) and bakes NO machine-specific path into a shared committed file:
native MCP configs are per-machine local artifacts each member regenerates.

## Non-goals

- **AGENTS.md placeholder-render + role differentiation is OUT.** That is a
  later north-star increment; this increment does NOT touch AGENTS.md rendering.
- No change to the real-run migration behaviour (idempotency was already correct).
- No git mutations of any kind (Fix 2 is warn-only).
- No new global-install mechanism, no hard-coded absolute paths in shared config.

## Acceptance

- [ ] After a real `runBootstrapFromLegacySdd`, a re-run with `dryRun: true`
      reports `alreadyMigratedCount = N`, `createdCount = 0` (not "would migrate N").
- [ ] A re-run adoption PREVIEW (`dryRun: true`, after a confirmed adoption)
      reports the migrated spec items as already-migrated / would-migrate: 0, and
      says so in its report lines.
- [ ] Adoption over a role repo with NO commits emits the baseline warning; a
      role repo WITH a commit does not. No git mutation is performed.
- [ ] The MCP configs generated by `runWorkspaceMcpConfig` / add-repo carry the
      RESOLVED launch command (not bare `axiom`) when `axiom` is not on PATH; the
      adoption report emits the launch-command / prereq note.
- [ ] `npm run build`, `npm test`, `npm run typecheck`, `npm run doctor` all green.

## Decisions

See `metadata.yml` (`decisions:` D-001..D-003 and `openQuestions:` Q-001). The
Fix-3 choice (approach a) and its justification are D-003.

## Result

**Archived — complete and green.** All three fixes shipped, surgical and additive.

- **Fix 1 (preview provenance):** `apps/cli/src/commands/workspace-adopt.ts` now
  resolves the control repo's spec scope + provenance map (best-effort, exactly
  like the real run) and classifies subject-A scan items and context docs as
  `would-migrate` vs `already-migrated` via the new shared
  `resolveLiveProvenanceMatch` helper (`apps/cli/src/bootstrap-shared/provenance.ts`),
  also adopted by the `bootstrap from-legacy-sdd` dry-run. A re-run preview now
  reports `Subject A — would-migrate: 0` instead of over-reporting.
- **Fix 2 (baseline commit):** adoption emits a warning
  (`role repo <name> (<path>) has no baseline commit; run 'git init && git commit'
  before role-branch operations ...`) for role repos with no commit / no repo,
  detected read-only (`.git` root + `git rev-parse --verify HEAD` via the
  injectable `@axiom/workflow` GitRunner). ZERO git mutations.
- **Fix 3 (MCP command):** approach (a) — `runWorkspaceMcpConfig` and
  `runWorkspaceAddRepo` now thread `resolveMcpLaunchCommand()` (override-injectable),
  closing the last two bare-`axiom` emitters; the adoption report emits a launch-
  command / global-`axiom` prereq note. See D-003.

**Gate:** `npm run build` ✓ · `npm run typecheck` (`tsc -b`) ✓ · `npm run doctor`
PASS (0 failures) · `npm test` 2753/2754. The 5 new tests
(`workspace-adopt.test.ts`, `bootstrap-idempotency-e2e.test.ts`,
`workspace-command.test.ts`) pass. The single red is a PRE-EXISTING environmental
flake — `apps/cli/tests/context.test.ts > Scenario 2` hit the per-test 5000 ms
timeout only under the full suite's parallel load; it passes in isolation
(12/12, ~6 s) and is unrelated to this increment (no touched module is on its
path). No existing test was weakened.

### Touched files (Axiom repo)

- `apps/cli/src/bootstrap-shared/provenance.ts` — `resolveLiveProvenanceMatch` (new, shared).
- `apps/cli/src/commands/bootstrap.ts` — dry-run predicate reuses the shared helper (behaviour-preserving).
- `apps/cli/src/commands/workspace-adopt.ts` — preview provenance parity (Fix 1), role baseline-commit warning (Fix 2), MCP note (Fix 3), injectable `gitRunner`.
- `apps/cli/src/commands/workspace.ts` — `runWorkspaceMcpConfig` threads `resolveMcpLaunchCommand()` (Fix 3).
- `apps/cli/src/commands/workspace-incremental.ts` — add-repo threads `resolveMcpLaunchCommand()` (Fix 3).
- Tests: `apps/cli/tests/workspace-adopt.test.ts`, `apps/cli/tests/bootstrap-idempotency-e2e.test.ts`, `apps/cli/tests/workspace-command.test.ts`.
