# FIX-E — unblock `axiom upgrade` / `sync` on adopted projects

## Goal

Make `axiom upgrade` (and its default post-`sync`) work **out-of-the-box on an adopted
project**. Today, adopting a foreign project via `axiom workspace setup` / `axiom workspace
adopt` leaves two gaps that make the version-update flow fail unless the operator hand-holds
it with `--no-sync --no-doctor`. This increment closes both, spec-first, without changing any
public signature.

## The two defects (confirmed on a version-update test of an adopted KVP25 project)

### Defect 1 — Adoption never materializes `init.json` → `axiom upgrade` is gate-blocked

`axiom upgrade` runs inside the orchestrator gate `upgrade-command`, whose precondition
`hasInitJson` (`@axiom/orchestrator`, `state-machine.ts`) requires
`<controlRepoRoot>/.axiom-state/<name>/init.json` (where `<name>` is
`resolveProject().name` === `axiom.yaml#name`). If it is missing the gate refuses with
`missing-init-json`.

`axiom init` writes that file natively (`init.ts`, `InitRecord` = `{ profileTriple,
createdAt, version }`). The ADOPTION / `axiom workspace setup` path (`workspace-setup.ts`'s
`runWorkspaceSetup`, the engine `runWorkspaceAdopt` also delegates to) writes `axiom.yaml`,
topology, `.axiom-state/<projectId>/workspace.json`, adapters, skills, MCP — but **never
`init.json`**. So `axiom upgrade` (even `--dry-run`) fails on a freshly adopted project.

**Root cause:** `runWorkspaceSetup` has no init.json materialization step. There is precedent
for the fix: `member-install.ts`'s `ensureInitJsonForJoin` self-heals a missing init.json for
the join case.

### Defect 2 — Post-migration `sync` hard-fails on adopted topology (`template-missing`)

Default `axiom upgrade` runs a post `sync`. On an adopted project that `sync` fails with
`adapterGenerationFailed: true — kind="template-missing" — … axiom.spec/templates/agents-md-template.md`,
which forces `executeUpgrade` to roll back — so only `axiom upgrade --no-sync --no-doctor`
completes.

**Root cause:** `sync.ts`'s `materializeAdapterOutputs` calls `generateOpencodeConfig` /
`generateClaudeCodeConfig` **without** a `templateContent` argument. The generators then read
`agents-md-template.md` from ON DISK (`<root>/axiom.spec/templates/agents-md-template.md`) and
hard-fail with `template-missing` when the adopted spec repo does not have it there. Axiom
already bundles that template in code (`workspace-adapter-templates.ts`'s `AGENTS_MD_TEMPLATE`,
used by `generateWorkspaceAdapters`), so the on-disk read is avoidable.

## Scope

1. **Defect 1 fix** — In `runWorkspaceSetup` (`workspace-setup.ts`), after `workspace.json` is
   written, materialize a faithful `init.json` at `<control>/.axiom-state/<name>/init.json`
   using the same `InitRecord` shape `axiom init` writes (`profileTriple` derived from
   `spec.profile` / `spec.overlay` / first selected adapter; `createdAt`; `version: '0.1.0'`).
   Idempotent (skip if present); guarded to only write when the control repo's `axiom.yaml`
   belongs to this project. Because `runWorkspaceAdopt` and `repo add`/`role add`
   (`workspace-incremental.ts`) all delegate to `runWorkspaceSetup`, they inherit the fix.
2. **Defect 2 fix** — In `sync.ts`, resolve the AGENTS.md template content once with **on-disk
   precedence → bundled fallback**: prefer `<root>/axiom.spec/templates/agents-md-template.md`
   when it exists and is readable, else use the bundled `AGENTS_MD_TEMPLATE`. Pass the resolved
   string as `templateContent` to `generateOpencodeConfig` / `generateClaudeCodeConfig`. The
   self-hosted Axiom repo keeps reading its own on-disk template (byte-identical to the bundled
   constant), so its behavior is unchanged.
3. **Tests** — Defect 1 unit/integration (init.json exists with correct shape after
   setup/adopt; `axiom upgrade --dry-run` passes the gate; idempotent). Defect 2 unit
   (bundled-fallback succeeds when the on-disk template is absent; on-disk template still takes
   precedence when present). Real e2e in a throwaway adopted sandbox.

## Non-goals (cataloged separately, OUT of this increment)

- **Operator-invokable rollback restore** — surfacing the pre-upgrade checkpoint restore as a
  first-class operator command is out of scope here.
- **Cross-repo upgrade fan-out** — running `axiom upgrade` across every repo of a multi-repo
  workspace in one shot is out of scope here.

## Acceptance

- After `axiom workspace setup` / `axiom workspace adopt` of a multi-repo project,
  `<control>/.axiom-state/<name>/init.json` EXISTS with the `axiom init` shape
  (`profileTriple` + `createdAt` + `version`).
- `axiom upgrade --dry-run` from the control repo of an adopted project **passes the gate**
  (no `missing-init-json`).
- Re-adopting / re-running setup does NOT clobber an existing valid `init.json` (idempotent).
- `axiom sync` on an adopted project whose spec repo lacks
  `axiom.spec/templates/agents-md-template.md` **succeeds** (uses the bundled template) and
  writes its adapter outputs + `last-sync.json`.
- When an on-disk `agents-md-template.md` IS present it still takes precedence.
- Default `axiom upgrade` (WITH sync + doctor) completes on an adopted project **without a
  forced rollback**.
- Gate stays green: `npm run build` → `npm test` → `npm run typecheck` → `node
  apps/cli/dist/index.js doctor` (modulo the known heavy-load I/O test-timeout flake).

## Result

**Done — green.** Both defects fixed, spec-first, no public-signature changes.

- **Defect 1 fix** — `apps/cli/src/commands/workspace-setup.ts` (`runWorkspaceSetup`, new step 6b)
  materializes `<control>/.axiom-state/<name>/init.json` after `workspace.json`, using the same
  `InitRecord` shape as `axiom init` (`profileTriple` from `spec.profile`/`spec.overlay`/first
  adapter, `createdAt`, `version: '0.1.0'`). Idempotent (skips if present) and guarded to the
  control repo's own `axiom.yaml` (no-clobber). `runWorkspaceAdopt` and `repo add`/`role add`
  inherit it by delegation.
- **Defect 2 fix** — `apps/cli/src/commands/sync.ts` adds `resolveAgentsMdTemplateContent(rootPath)`
  (on-disk `<root>/axiom.spec/templates/agents-md-template.md` if present/readable, else the bundled
  `AGENTS_MD_TEMPLATE`) and passes the result as `templateContent` to `generateOpencodeConfig` /
  `generateClaudeCodeConfig`. The exact reader that hard-failed was the generators' `readTemplate`
  on the default disk path (invoked because `materializeAdapterOutputs` supplied no
  `templateContent`).
- **Build ownership** — `workspace-adapter-templates.ts` was moved to `@axiom/cli-commands`
  ownership (it now compiles `sync.ts`) and re-exported from the barrel; `workspace-adapters.ts`
  imports the constants from `@axiom/cli-commands`. This respects the single-ownership build rule.
- **Tests** — `sync.test.ts` Scenario 7 rewritten (bundled fallback SUCCEEDS) + new Scenario 7b
  (on-disk precedence). `workspace-setup.test.ts` gained two FIX-E tests (init.json shape +
  idempotency).
- **Real e2e** (throwaway adopted workspace via `axiom workspace setup --adopt-sdd/--adopt-spec`):
  init.json materialized; `axiom upgrade --dry-run` passes the gate (exit 0, no `missing-init-json`);
  `axiom sync` with NO on-disk template succeeds via the bundled template (`generatedFilesCount: 2`,
  `.opencode/AGENTS.md` written); default `axiom upgrade` (sync + doctor) completes (exit 0,
  `syncRun: true`, `doctorRun: true`) with no forced rollback.
- **Gate** — `npm run build` clean · `npm test` 2785 passed / 0 failed (274 files; +3 vs the 2782
  baseline) · `npm run typecheck` clean · `doctor` PASS (0 failures). No `Test timed out in 5000ms`
  failures.
