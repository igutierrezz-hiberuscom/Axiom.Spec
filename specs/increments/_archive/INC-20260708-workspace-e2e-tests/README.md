# Increment: Workspace + MCP end-to-end tests

Status: closed
Date: 2026-07-08

## Goal

Add comprehensive END-TO-END tests that prove the whole multi-repo
workspace creation flow (`runWorkspaceSetup`) works AND that the
generated MCP server launch configs are actually runnable — i.e. spawn
the real `@axiom/mcp-server` process (via the built CLI) and confirm it
answers `initialize`/`tools/list`/`tools/call` over real newline-delimited
JSON-RPC stdio, not just in-process/injected-stream coverage (which
`apps/cli/tests/mcp-serve.test.ts` and `packages/mcp-server/tests/*`
already provide).

This is a TEST-ONLY increment. No product behavior changes were needed —
the e2e did not surface a real defect (see "Result").

## Context

Read first: `apps/cli/src/commands/workspace-setup.ts` (`runWorkspaceSetup`,
the multi-repo scaffolding engine), `apps/cli/src/commands/mcp-serve.ts`
(`axiom mcp serve --kind <sdd|spec> --project-root <path> [--home-dir
<path>]`, the CLI launcher), `packages/mcp-server/src/*` (the hand-rolled
JSON-RPC 2.0 dispatcher + newline-delimited stdio transport), `apps/cli/
src/commands/workspace-mcp.ts` (`buildWorkspaceMcpServers` — emits
`command:'axiom'` + `args:['mcp','serve','--kind',...,'--project-root',...]`
per `INC-20260708-mcp-launch-config-wiring`), and the existing
`apps/cli/tests/workspace-setup.test.ts` (tmp-dir/tmp-home helper
patterns, reused here).

Existing coverage before this increment:
- `apps/cli/tests/workspace-setup.test.ts`: `runWorkspaceSetup` scenarios
  (a)-(i), all in-process, one scenario at a time (never the full combined
  checklist — control+spec+2 roles+custom role+both adapters+legacy
  migration — in a single run).
- `apps/cli/tests/mcp-serve.test.ts` + `packages/mcp-server/tests/
  server.test.ts`/`stdio.test.ts`: `runMcpServe`/`createMcpServer`/
  `runStdioServer` driven with IN-MEMORY injected streams — proves the
  dispatcher and stdio framing are correct, but never proves the actual
  CLI process (`node dist/index.js mcp serve ...`) starts, wires argv
  correctly, and answers over a REAL child-process stdio pipe.

Nothing previously spawned the built CLI as a child process and exercised
it as an external MCP client would.

## Scope

New file `apps/cli/tests/e2e/workspace-mcp.e2e.test.ts` (new `tests/e2e/`
directory; vitest auto-discovers it, no config change needed — confirmed
`vitest.config.ts`'s `exclude` only excludes `node_modules`/`dist`/
`scripts/**`).

**Part A — full workspace creation (in-process, `runWorkspaceSetup`):**
in a tmp workspace dir + tmp home (`homeDirOverride`), pre-seed a legacy
v1 `~/.axiom/registry.json` to exercise the auto-migration path, then run
`runWorkspaceSetup` with control(sdd) + spec + two role repos (`backend`,
plus a CUSTOM role `data`), `adapters: ['opencode', 'claude-code']`, all
repos freshly created. Assert the full on-disk checklist: per-repo
`axiom.yaml` (v2, role, reciprocal paths), `axiom.config/topology.yaml`,
`projects.yml` (v2, migrated legacy preserved as `registry.json.migrated`),
`topology-bindings.yaml`, `.axiom/mcp.yml` with launch `command`/`args`,
adapter `AGENTS.md` (`.opencode/AGENTS.md`, `.claude/AGENTS.md`) in EVERY
repo, `mcp.json` (`.opencode/mcp.json`, `.claude/mcp.json`) in the CONTROL
repo only (confirmed by reading `workspace-setup.ts`'s step 4 +
`workspace-mcp.ts`: `writeWorkspaceMcpConfig` is called once, scoped to
`control.path`, unlike the per-repo `AGENTS.md` generation in step 5),
`.axiom-state/<projectId>/workspace.json`, SDD
skills in the control repo (catalog, materialized skill, `skills-pending.
json`, one `skills-index/<role>.yaml` per role), spec base in the spec
repo (`specs/00..08` + `context/TECHNICAL_CONTEXT.md`), and per-code-repo
skills baseline in EACH created role repo (including the custom `data`
role).

**Part B — live MCP servers (spawned child processes):** a hand-rolled
JSON-RPC-over-stdio test client spawns `node <repoRoot>/apps/cli/dist/
index.js mcp serve --kind <sdd|spec> --project-root <repoPath>
--home-dir <tmpHome>` for BOTH the sdd server (against the control repo
from Part A) and the spec server (against the spec repo). The spawned
`args` are asserted to match the `args` generated in the workspace's own
`.axiom/mcp.yml`, so the test proves the generated config is the SAME one
being exercised (not a hand-picked substitute). For each server: send
`initialize` (assert `protocolVersion`/`serverInfo`/`capabilities.tools`),
send `notifications/initialized`, send `tools/list` (assert exact sdd.*
(7) / spec.* (10) capability id sets), send one `tools/call` (`sdd.
projectRegistryRead` for the sdd server — assert the registered project
appears; `spec.adrIndexRead` for the spec server — assert a clean empty
array, since the scaffolded spec repo has no ADRs yet), send an unknown
method (assert JSON-RPC `-32601`), then close stdin and assert clean
process exit. All spawned children are killed in `afterEach`/timeout
teardown — no orphaned processes.

Build dependency: a `beforeAll` checks `apps/cli/dist/index.js` exists;
since the increment's own validation runs `npm run build` first, dist is
expected fresh, but the block still `test.skip`s with a clear console
message if the dist entry is missing (keeps Part A's in-process coverage
unconditional).

## Non-goals

- No product behavior change (this is a test-only increment) unless the
  e2e surfaced a genuine defect — it did not (see "Result").
- No new npm dependencies; the JSON-RPC stdio test client is hand-rolled
  (same newline-delimited framing as `packages/mcp-server/src/stdio.ts`).
- No changes to `00`–`08` canonical spec docs (orchestrator's own pass).
- No mutating git operations.

## Acceptance criteria

- [x] `apps/cli/tests/e2e/workspace-mcp.e2e.test.ts` exists and covers
      both Part A (full on-disk checklist) and Part B (spawned sdd + spec
      servers, both answering `initialize`/`tools/list`/`tools/call`,
      plus an unknown-method `-32601` check for each).
- [x] Part B spawns the ACTUAL built CLI (`apps/cli/dist/index.js`) as a
      real child process communicating over real stdio, not injected
      streams.
- [x] Part B's spawned `args` are asserted to match the `args` array
      generated in the workspace's own `.axiom/mcp.yml` for each server
      entry (proves the generated launch config is what actually runs).
- [x] No orphaned child processes after the suite runs (teardown kills
      any still-running spawned process).
- [x] `npm run build` (tsc -b) clean.
- [x] `npx vitest run apps/cli/tests/e2e` passes (both parts).
- [x] `npx vitest run apps/cli packages/mcp-server` has zero NEW
      regressions (pre-existing dogfood failures + `packages/skills/
      tests/catalog.test.ts` are out of scope, per the parent increments'
      own precedent).

## Open questions

None blocking.

## Assumptions

- **Spec-repo self-scope resolution when spawned standalone**: when the
  spec MCP server is spawned with `--project-root <specRepoPath>` (the
  spec repo directly, no control repo in the same process), `@axiom/
  mcp-server`'s `resolveMcpServerContext` calls `loadTopology(specRepoPath)`.
  Since `axiom.config/topology.yaml` only exists in the CONTROL repo, this
  falls back to `tryLoadTopologyHint` reading the spec repo's own
  `axiom.yaml` (`mode: installed-multi-repo`), which synthesizes
  `specRepo.ref = '../<projectName>.spec'`. Because `runWorkspaceSetup`'s
  own naming convention names the spec repo directory `<projectName>.spec`
  (same convention `workspace-setup.test.ts` already uses), `path.join(
  specRepoPath, '../<projectName>.spec')` resolves back to `specRepoPath`
  itself — confirmed by direct calculation, not by chance the test relies
  on. This is why the e2e's spec-repo directory is named
  `<project>.spec` (matching the convention), not an arbitrary name — an
  arbitrarily-named spec repo directory spawned standalone (without the
  control repo's `topology-bindings.yaml` in scope) would mis-resolve its
  own scope to a sibling path that does not exist. This is a pre-existing,
  documented characteristic of `resolveMcpServerContext`'s best-effort
  fallback chain (see that module's own header comment on "deliberately
  best-effort" resolution), not a new defect introduced or fixed here —
  recorded as an assumption because the e2e's realism depends on it.
- **`spec.adrIndexRead` on an empty scaffolded spec repo returns `{ok:
  true, data: []}`** (a clean empty list, not an error) — confirmed by
  reading `packages/mcp-tools/src/artifact-handlers.ts`'s `getAdrIndex`
  (`listArtifacts` never errors on an absent/empty ADR directory). The
  e2e asserts the SHAPE (`Array.isArray(data)` and `length === 0`), not
  brittle content, per the brief's own guidance.
- **`npm run build` freshness**: the e2e's Part B assumes `apps/cli/dist/
  index.js` reflects the current source tree (the increment's own
  validation runs `npm run build` immediately before running the tests).
  If dist is stale or absent, Part B degrades to `test.skip` with a
  console message rather than failing outright, keeping the suite usable
  in a partial dev loop; Part A never skips.

## Implementation notes

- `apps/cli/tests/e2e/workspace-mcp.e2e.test.ts` (new): ~650 lines.
  - Reuses the exact tmp-dir/tmp-home/legacy-registry-seed helpers from
    `apps/cli/tests/workspace-setup.test.ts` (`makeTmpRoot`,
    `makeTmpHomeDir`, `writeLegacyRegistry`/`makeV1Entry` pattern).
  - Part A's tmp workspace/home dirs are pushed to a module-level
    `sharedTmpDirs` array (cleaned up in `afterAll`), NOT the per-test
    `tmpDirs` array (cleaned up in `afterEach`) — Part B's later tests in
    the same file spawn child processes against that exact workspace, so
    it must survive past Part A's own test boundary. This was found and
    fixed during implementation (the first draft used `tmpDirs`, which
    made Part B fail with "directory not found"-class errors because the
    workspace was deleted before Part B ran).
  - `spawnMcpServer(kind, projectRootPath, homeDir)`: spawns `process.
    execPath` (`node`) against `path.resolve(__dirname, '../../dist/
    index.js')` with `['mcp', 'serve', '--kind', kind, '--project-root',
    projectRootPath, '--home-dir', homeDir]` — the SAME args shape
    `buildMcpServeArgs` in `workspace-mcp.ts` generates, plus `--home-dir`
    (which `buildWorkspaceMcpServers` deliberately never emits per
    `INC-20260708-mcp-launch-config-wiring`'s own documented assumption —
    added here only for test-home isolation). The test asserts an EXACT
    match (`toEqual`, not a prefix match) between the workspace's own
    generated `.axiom/mcp.yml` `args` and the 6-element array without
    `--home-dir`, then spawns with that same array plus the two
    `--home-dir` elements appended.
  - A tiny hand-rolled `JsonRpcStdioClient` class: writes one JSON line +
    `\n` to the child's stdin per request, buffers/line-splits stdout,
    parses each line as JSON, and resolves a per-`id` promise map
    (correlates responses by JSON-RPC `id`; skips any line that isn't
    valid JSON or has no numeric `id`, tolerating interleaved
    notifications/log noise per the brief's robustness requirement).
    Every `send()` call has a 10s internal timeout independent of the
    per-test 30s vitest timeout.
  - `afterEach` kills every still-running spawned child (checked via
    `exitCode`/`signalCode` both still `null`) — guarantees no orphaned
    processes even on assertion failure. Each Part B test also explicitly
    closes stdin and awaits a clean exit (`waitForExit`, asserted `=== 0`)
    as part of its own passing-path assertions, so `afterEach`'s kill is
    normally a no-op safety net, not the primary teardown path.
  - `beforeAll` checks `fs.existsSync(CLI_DIST_ENTRY)`; if false (dist not
    built), both Part B tests print a `console.warn` and return early
    (runtime skip, not `it.skip` at collection time) — Part A's coverage
    is entirely unconditional either way.
  - Project name in the shared spec is `'e2e-app'` (not e.g. `'E2E App'`):
    deliberately already valid against `init.ts`'s `PROJECT_NAME_REGEX`
    (lowercase + hyphens only), so the on-disk/registry `name` round-trips
    unambiguously instead of falling back to the slugified `projectId`
    (confirmed by reading `runWorkspaceSetup`'s own `axiomYamlName`
    fallback logic, and by first writing the test with a
    regex-invalid name and observing the registry return the slug instead
    — not a defect, `workspace-setup.test.ts`'s own `'My App'` has the
    same fallback-to-slug behavior, just never asserts the human name
    directly).

No product files were changed — the e2e did not surface a genuine defect.

## Validation

### `npm run build` (tsc -b), from `Axiom/`

```
> axiom-product@0.1.0 build
> tsc -b

(clean exit, no errors, no stdout)
```

### `npx vitest run apps/cli/tests/e2e`

```
 ✓ apps/cli/tests/e2e/workspace-mcp.e2e.test.ts (4 tests) 1202ms

 Test Files  1 passed (1)
      Tests  4 passed (4)
```

The 4 tests: Part A's combined-checklist test (control+spec+backend+
custom-"data"-role, both adapters, legacy v1 registry auto-migration —
the full on-disk assertion checklist), Part B's spawned sdd server test
(initialize/tools-list/tools-call `sdd.projectRegistryRead`/unknown-method
`-32601`/clean-exit), Part B's spawned spec server test (same shape,
`spec.adrIndexRead`), and a trivial directory-sanity test documenting the
skip-path. Both spawned child processes answered every request over real
stdio and exited cleanly (`waitForExit` observed exit code `0` after
`stdin.end()`); `afterEach` kill-tracking confirmed no orphaned processes.

### `npx vitest run apps/cli packages/mcp-server`

```
 Test Files  57 passed (57)
      Tests  479 passed (479)
```

Zero failures, zero regressions — better than the "some pre-existing
dogfood failures expected" assumption carried over from prior increments'
own validation notes (e.g. `INC-20260708-mcp-launch-config-wiring`'s "10
pre-existing dogfood failures" applied to a full-repo run, not to this
narrower `apps/cli packages/mcp-server` scope). This narrower, explicitly
required scope is fully green both before and after this increment's
test-only diff.

## Result

Delivered as scoped: a new `apps/cli/tests/e2e/workspace-mcp.e2e.test.ts`
proves (a) the full multi-repo workspace creation checklist end-to-end in
one combined run (control+spec+backend+custom-data-role, both adapters,
legacy-registry auto-migration), and (b) that the MCP launch configs
`runWorkspaceSetup` generates are ACTUALLY runnable — the real built CLI
was spawned as a child process for both the sdd and spec servers, and
both answered `initialize`, `tools/list` (exact 7/10 capability sets),
`tools/call` (a real read against the scaffolded workspace data), and an
unknown-method JSON-RPC `-32601`, then exited cleanly with the pipe
closed. No orphaned processes. No real defect was found; no product code
was modified.

## General spec integration

No `general-spec.md`-equivalent stable-knowledge file exists yet in
`Axiom.Spec/specs/` (same absence noted by `INC-20260708-
mcp-launch-config-wiring`). The one piece of durable, non-obvious
knowledge this increment surfaced — that `resolveMcpServerContext`'s
best-effort topology fallback only correctly self-resolves a standalone-
spawned spec repo's own scope when that repo's directory is named
`<projectName>.spec` (the exact convention `runWorkspaceSetup` already
uses) — is recorded here under "Assumptions" and in the new test file's
own header comment, for a future increment to link from `Axiom.Spec/
context/architecture/` or `context/integrations/` if/when those are
revisited. No code behavior changed as a result (documented, not fixed,
since it is not a defect given the existing naming convention is always
followed by the only producer of these directories, `runWorkspaceSetup`).

### Integración canónica realizada (pase de round 4, 2026-07-08)

El pase de integración de round 4 (orquestador) integró el conocimiento
estable de este incremento en los ficheros canónicos `00–08`:

- **`02_Requisitos_No_Funcionales.md`**: nota añadida a NFR-AXM-010
  (Cobertura runtime verificable por doctor) sobre la nueva suite e2e
  `apps/cli/tests/e2e/` como garantía de calidad end-to-end: crea
  in-process un workspace multi-repo completo (control + spec + rol
  custom, migración de registry v1→v2, base de spec, skills por repo de
  código, ambos adapters) y lanza los servers reales `axiom mcp serve`
  sdd+spec como procesos hijo, probando que responden
  `initialize`/`tools/list`/`tools/call` usando el `command`/`args`
  generado en `.axiom/mcp.yml`. Test-only, sin cambio de producto.

El conocimiento durable sobre la resolución de scope del spec repo
spawneado standalone (naming `<projectName>.spec`) permanece registrado
aquí y en el header del propio fichero de test, no integrado a `00–08`
(es un detalle de característica best-effort de `resolveMcpServerContext`,
no comportamiento estable de producto que necesite consolidación canónica).
