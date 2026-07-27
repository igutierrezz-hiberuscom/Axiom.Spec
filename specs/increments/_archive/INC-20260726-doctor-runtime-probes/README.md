# Increment: Doctor runtime probes (real MCP liveness + tool functional checks, opt-in)

Status: closed
Date: 2026-07-26

## Goal

Today `axiom doctor` (the synchronous `runDoctorChecks`,
`packages/doctor/src/index.ts:99`) verifies CONFIGURATION only:

- TC-007 (`runMcpBindingsCoherenceCheck`) only checks that
  `mcp-manifest.yaml` DECLARES the `sdd`/`spec` MCP servers
  (`projectBinding: required`) — it never spawns or pings anything.
- TC-004/TC-005 (`runToolchainP0/P1CoverageCheck`) only check that a
  tool-detection MARKER directory exists (`detectToolState`) — never that
  the underlying binary actually runs.

Add ADDITIVE, OPT-IN, NON-FAILING runtime probes so a user can ask doctor to
actually verify "MCPs levantados" (MCP servers reachable) and "herramientas
funcionando" (tools respond) — without touching the existing synchronous
check tree, its ~180 tests, or the launcher doctor-gate's exit-code
semantics.

## Context

- `packages/toolchain/src/probe.ts` already has the exact precedent this
  increment generalizes for MCP: `resolveProbeCommand`/`probeToolInstalled`/
  `detectToolStateWithProbe` — an OPTIONAL, ASYNC, best-effort real probe on
  top of the SYNCHRONOUS marker-only `detectToolState`, never upgrading a
  state unless the probe positively confirms it, never throwing.
  `axiom toolchain show/validate` already wire an injectable `probeFn`/
  `probeTimeoutMs` through `detectAllToolsWithProbe` (`apps/cli/src/commands/
  toolchain.ts`).
- `runProviderSelectionChecksAsync` (`packages/doctor/src/checks.ts:3180`) is
  a smaller precedent for the SAME pattern already living in `@axiom/doctor`:
  an async variant of a sync check, exported but NOT wired into
  `runDoctorChecks`, explicitly reserved for "a future caller that can pay
  the cost of a real spawn (e.g. `axiom doctor --deep`)".
- `resolveProbeCommand` deliberately returns `null` for `rtk`/`caveman` (no
  known local binary contract — `rtk` is invoked ONLY from a skill, never a
  standalone process, per `INC-20260724-rtk-skill-invoked`). This increment
  must not fabricate a probe for those.
- The real launch `command`/`args` for the two Axiom-managed MCP servers
  (`sdd-mcp-server`, `spec-mcp-broker`) are written into the project's
  canonical `.axiom/mcp.yml` by `writeWorkspaceMcpConfig`
  (`apps/cli/src/commands/workspace-mcp.ts`, `INC-20260708-mcp-launch-
  config-wiring`): `axiom mcp serve --kind <sdd|spec> --project-root <path>`.
  The reader, `loadMcpProjectConfig` (`@axiom/user-workspace/src/mcp-
  config.ts`), is a lightweight PACKAGE (not an app), so `@axiom/doctor`
  (also a package) can depend on it without violating the repo's
  package-must-not-depend-on-app boundary (`runDogfoodingBoundaryChecks`).
- `@axiom/mcp-server`'s `createMcpServer` (`packages/mcp-server/src/
  server.ts`) answers the real MCP `initialize` JSON-RPC method for every
  server kind, including `sdd`/`spec` — so a REAL handshake (not just "the
  process started") is genuinely reachable from a probe, not just a
  simplified "did it stay alive" check.
- `@axiom/providers`'s `createStdioMcpClient` (`packages/providers/src/
  stdio-mcp-client.ts`) is an EXISTING, reusable local-stdio-MCP-client
  transport (spawn + newline-delimited JSON-RPC 2.0 + `initialize()` +
  `close()`, never throws synchronously) built for AB4/AB5's
  `ProviderClient`s. `@axiom/doctor` already depends on `@axiom/providers`
  (for PS-001's `readEnabledProviders`/`buildProjectProviderRegistry`), so
  reusing `createStdioMcpClient` here required zero new cross-package
  dependency risk — just a genuine, real `initialize` handshake instead of
  the "acceptable simpler fallback" (spawn-and-see-if-it-stays-alive) the
  brief allowed as a fallback.
- This repo (`Axiom`, self-hosted layout) does not itself have a
  `.axiom/mcp.yml` — only `axiom.config/mcp-manifest.yaml` (the
  capability-catalog TC-007 reads, now additionally carrying a `server:`
  field naming `sdd-mcp-server`/`spec-mcp-broker`/`axiom-mcp-broker`, but
  still no `command`/`args`). Running `axiom doctor --deep` in THIS repo
  therefore currently reports an honest `warn` for both `TC-019-*` checks
  ("no launch command declared") — an accurate reflection of this repo's
  real state, not a defect of the new checks.

## Scope

- `packages/doctor/src/checks.ts`: exported the existing private `pass`/
  `warn`/`skip` helpers (added the `export` keyword only) so the new
  module can build `DoctorCheck` values identically instead of duplicating
  them. `fail` stays private/unexported — the new checks must never use it.
- `packages/doctor/src/deep-checks.ts` (new): the two new async, opt-in
  checks + their injectable prober types:
  - `runToolchainFunctionalProbeCheck(resolution, opts?)` → `TC-018-<toolId>`
    per tool declared in the project's `toolchain.yaml` (same enumeration
    `loadToolchain` gives TC-004/TC-005).
  - `runMcpServerLivenessCheck(resolution, opts?)` → `TC-019-<serverId>` per
    Axiom-managed MCP server (`sdd-mcp-server`, `spec-mcp-broker`).
  - `CATEGORY_RUNTIME_PROBES = 'runtime-probes'` — a NEW, dedicated
    category so a report/CI consumer can trivially filter "checks that
    actually touched a real process" from the rest.
  - `ToolchainFunctionalProbeOptions { probeFn?, timeoutMs? }` (mirrors
    `@axiom/toolchain`'s `DetectWithProbeOptions`), `McpServerLivenessOptions
    { probeFn?, timeoutMs? }`, `McpProbeFn`, `McpProbeResult`,
    `DEFAULT_MCP_PROBE_TIMEOUT_MS = 3_000`.
- `packages/doctor/src/index.ts`: re-exports the new module's public
  surface; adds `runDoctorChecksDeep(resolution, opts?)` (async) —
  `runDoctorChecks(resolution)` (unchanged) + the two new checks, `await`ed
  and merged via the existing `buildReport`.
- `packages/doctor/package.json` + `tsconfig.json`: new dependency/project
  reference on `@axiom/user-workspace` (for `loadMcpProjectConfig`).
- `apps/cli/src/index.ts`: `axiom doctor --deep` — without the flag,
  behavior is byte-for-byte unchanged (still `runDoctorChecks`, still sync);
  with it, `await runDoctorChecksDeep(resolution)`. `--json` composes with
  `--deep` unchanged (same `formatReport`/`JSON.stringify` branch, same
  exit-code rule: `report.summary.failed > 0`).
- `apps/cli/src/commands/app-launcher.ts` (`apiGetLauncherDoctor`) +
  `apps/cli/src/commands/app-api.ts` (`RouteMatch`, `matchRoute`, the
  `projects.launcherDoctor` case): the launcher doctor-gate endpoint
  (`GET /api/projects/:id/launcher/doctor`) now accepts `?deep=1`/`?deep=true`
  to opt into `runDoctorChecksDeep`; the DEFAULT (no query param) stays the
  fast synchronous `runDoctorChecks` path, so the gate is never slowed down
  or made flaky for existing callers.
- Tests: `packages/doctor/tests/deep-checks.test.ts` (new, 16 tests — both
  checks + `runDoctorChecksDeep`, all with injected fakes, zero real
  spawns); `apps/cli/tests/launcher-doctor.test.ts` (1 new test for
  `?deep=1`, using the SAME zero-declared-tools/zero-mcp.yml fixture so the
  probes never spawn a real process end-to-end either).

## Non-goals

- Changing anything about the existing synchronous `runDoctorChecks`, its
  check list, its ~180 existing tests, or the launcher gate's DEFAULT
  behavior — all confirmed byte-for-byte unchanged.
- Making the new checks ever produce `fail` — by design they can only add
  `pass`/`warn`/`skip`; `summary.failed` after `--deep` is always identical
  to the sync report's `failed` count.
- A full MCP capability negotiation/tools-list round trip — the liveness
  probe performs exactly the `initialize` handshake (the real, minimal
  "is this MCP server alive and speaking the protocol" signal), not
  `tools/list`/`tools/call`.
- Fabricating a probe contract for `rtk`/`caveman` (or any tool
  `resolveProbeCommand` returns `null` for) — `TC-018-rtk`/`TC-018-caveman`
  always `skip` with an explicit "skill-invoked / no binary probe" note,
  per `INC-20260724-rtk-skill-invoked`'s deliberate, unchanged design.
- A CLI-level (spawn-based) test for `axiom doctor --deep` itself — `doctor`
  is registered inline in `apps/cli/src/index.ts` (an entrypoint file with a
  top-level `program.parse(process.argv)`), which no existing test harness
  imports or spawns for unit testing, and there is no pre-existing "doctor
  CLI test harness" that supports fake-prober injection. Covered instead via
  (a) the `runDoctorChecksDeep` unit tests (exact same code path the CLI
  action calls) and (b) a real HTTP-level test of the launcher's `?deep=1`
  endpoint (`apps/cli/tests/launcher-doctor.test.ts`), which exercises the
  full `matchRoute` → `apiGetLauncherDoctor` → `runDoctorChecksDeep` chain
  end-to-end without ever spawning a real process (the fixture project has
  no declared tools/`.axiom/mcp.yml`, so both probes short-circuit to
  `pass`/`warn` without a real spawn).
- Rewiring `TC-007`/`TC-004`/`TC-005` themselves to call the new probes —
  they remain exactly as they were; the new checks are additive siblings,
  not replacements.
- Any change to `resolveProbeCommand`'s known-contract list (`serena`,
  `cmm`, `engram`) — reused verbatim from `@axiom/toolchain`.

## Acceptance criteria

- [x] `runDoctorChecks(resolution)` (the sync entry point) is unchanged:
      same check list, same call signature, same synchronous return type.
- [x] Two new async checks exist, both NEVER returning `fail`:
      `runToolchainFunctionalProbeCheck` (`TC-018-*`) and
      `runMcpServerLivenessCheck` (`TC-019-*`).
- [x] Both probes are fully injectable (`probeFn`/`McpProbeFn`) — no test
      spawns a real process.
- [x] `runDoctorChecksDeep` = `runDoctorChecks`'s checks + the two new
      checks, merged into one `DoctorReport`; `summary.failed` is always
      identical between the sync and deep reports.
- [x] `axiom doctor --deep` wires the new async path; `--json` still works;
      exit-code rule unchanged (`failed > 0` ⇒ exit 1).
- [x] The launcher doctor-gate endpoint accepts an OPT-IN `?deep=1`,
      defaulting to the fast sync path.
- [x] `npm run build` passes clean.
- [x] `npx vitest run packages/doctor packages/toolchain` — all pass
      (existing + new).
- [x] `npx vitest run apps/cli/tests` (doctor-referencing files, plus a full
      suite safety pass) — all pass, zero regressions.

## Open questions

None blocking.

## Assumptions

1. **The MCP liveness probe's canonical source is `.axiom/mcp.yml`
   exclusively** (via `loadMcpProjectConfig`, `@axiom/user-workspace`), NOT
   a synthesized fallback command. The brief named `.axiom/mcp.yml` as the
   source; a project that never wrote one (e.g. this repo's own
   self-hosted layout, or any project that hasn't run `axiom workspace
   setup`/`axiom member install`) gets an honest, actionable `warn` per
   managed server instead of a guessed `axiom mcp serve --kind <k>
   --project-root <resolution.rootPath>` command. Synthesizing a command
   the project never actually declared would blur "verified real launch
   config" with "Axiom's best guess at what it might be" — the same
   never-a-guess discipline `resolveProbeCommand` already applies to
   `rtk`/`caveman`.
2. **`@axiom/doctor` gained a new dependency, `@axiom/user-workspace`**
   (package.json + tsconfig path/reference), to read `.axiom/mcp.yml` via
   `loadMcpProjectConfig`. Verified safe: `@axiom/user-workspace` is a leaf
   package (`@axiom/core` + `@axiom/filesystem-truth` + `js-yaml` only, no
   app-side or heavy dependencies), and only the shape-only
   `loadMcpProjectConfig` is used — not `validateMcpProjectConfig` (which
   would additionally require a real user-registry lookup, irrelevant to a
   liveness probe).
3. **The `.axiom/mcp.yml` relative path (`path.join('.axiom', 'mcp.yml')`)
   is duplicated as an independent literal in `deep-checks.ts`**, rather
   than importing `apps/cli/src/commands/workspace-mcp.ts`'s
   `MCP_PROJECT_CONFIG_RELATIVE_PATH` constant. `@axiom/doctor` is a
   PACKAGE; `workspace-mcp.ts` lives in the CLI APP — importing app code
   from a package would invert the repo's own boundary rule this repo
   enforces via `runDogfoodingBoundaryChecks`. The convention itself is
   independently documented in `@axiom/user-workspace`'s `mcp-config.ts`
   header comment (`<specRoot>/.axiom/mcp.yml`), so this is a
   cross-referenced literal, not an invented one.
4. **The MCP liveness probe performs a REAL `initialize` handshake** (the
   brief's "preferred" option), not the "acceptable simpler fallback"
   (spawn-and-see-if-it-stays-alive). This was feasible within scope
   because `@axiom/providers`'s `createStdioMcpClient` already exists as a
   ready-made, tested, reusable local-stdio-MCP-client transport, and
   `@axiom/mcp-server`'s `createMcpServer` already answers `initialize` for
   every server kind — no new transport code was needed, only composing
   two already-existing pieces.
5. **ID scheme**: `TC-018-<toolId>` (one check per declared tool) and
   `TC-019-<serverId>` (one check per managed MCP server, `serverId` ∈
   `{sdd-mcp-server, spec-mcp-broker}`) — chosen over a single aggregate
   check per feature so a report consumer can see exactly which tool/server
   is unresponsive without parsing a combined evidence string, matching the
   existing convention of turning a claimed enumeration into 1 check-per-
   item wherever more actionable (there was no single existing 1-check-per-
   item precedent to follow verbatim in `@axiom/doctor` itself, but this
   mirrors `@axiom/toolchain`'s own per-tool `ToolDetection[]` shape).
6. **`CATEGORY_RUNTIME_PROBES = 'runtime-probes'`** is a NEW category
   (neither `toolchain` nor `memory`) — deliberately distinct from TC-004/
   TC-005 (`toolchain`) and TC-007/TC-008 (`memory`) so `--deep`'s
   additions are trivially filterable/groupable in `formatReport`'s
   per-category rendering and in any future CI consumer.
7. **The launcher's `?deep=1` opt-in was implemented** (the brief's Part 5
   was explicitly optional, "only if clean"). It was: `apiGetLauncherDoctor`
   had exactly one production call site (`app-api.ts`), no direct unit
   tests referencing it by name (only exercised through the HTTP-level
   `launcher-doctor.test.ts`), and `handleApiRequest` was already `async` —
   so making `apiGetLauncherDoctor` `async` + adding one optional `opts`
   parameter was a fully backward-compatible, low-risk change.

## Implementation notes

### `TC-018-<toolId>` — `runToolchainFunctionalProbeCheck`

1. Unresolved project → single `skip` (`TC-018`).
2. `loadToolchain(resolution.rootPath)` fails to parse → single `skip`
   (`TC-018`, evidence names the load error).
3. `manifest.tools.length === 0` → single informative `pass` (`TC-018`,
   "nothing to probe").
4. Otherwise, one check per tool, `id = TC-018-<toolId>`:
   - `resolveProbeCommand(tool)` → `null` (no known contract, e.g. `rtk`,
     `caveman`, `context7`, `autoskills`) → `skip` ("skill-invoked /
     instruction-driven; no spawn attempted").
   - Contract found → `await (opts?.probeFn ?? probeToolInstalled)(tool,
     resolution.rootPath, opts?.timeoutMs)`. `true` → `pass` (names the
     resolved command). `false` → `warn` (never `fail`). Thrown exception →
     caught, `warn` (evidence includes the exception message).

### `TC-019-<serverId>` — `runMcpServerLivenessCheck`

1. Unresolved project → single `skip` (`TC-019`).
2. For each of `sdd-mcp-server`/`spec-mcp-broker`:
   - `loadMcpProjectConfig(<rootPath>/.axiom/mcp.yml)` → find the
     `McpServerEntry` by `id`. Missing file, unreadable YAML, no matching
     entry, or an entry with no `command` → `warn` (names the server id,
     the resolved `mcp.yml` path, and the specific reason — file-not-found
     vs entry-absent vs no-command — plus an actionable pointer to `axiom
     workspace mcp`/`runWorkspaceSetup`).
   - Entry found with a `command` → `await (opts?.probeFn ??
     defaultMcpProbeFn)({command, args}, opts?.timeoutMs ??
     DEFAULT_MCP_PROBE_TIMEOUT_MS)`.
     - `defaultMcpProbeFn`: `createStdioMcpClient({command, args,
       timeoutMs})` → `client.initialize()` → `{ok: true, note: "...
       initialize ... serverInfo: name@version"}` on success; any thrown
       error (spawn failure, protocol error, timeout) → `{ok: false, note:
       "..."}`. Always `client.close()` in a `finally` (best-effort, never
       surfaces its own failure).
     - `ok: true` → `pass` (evidence: exact command + the real
       `serverInfo`). `ok: false` → `warn` (evidence: exact command + the
       real failure reason). Thrown exception from `probeFn` itself →
       caught, `warn`.

### `runDoctorChecksDeep`

```
runDoctorChecksDeep(resolution, opts?) =
  buildReport(
    [...runDoctorChecks(resolution).checks,
     ...(await runToolchainFunctionalProbeCheck(resolution, opts?.toolchainProbe)),
     ...(await runMcpServerLivenessCheck(resolution, opts?.mcpProbe))],
    resolution,
  )
```

Delegates to `runDoctorChecks` itself (not a re-listed copy of every
`runX(resolution)` call) so the two check lists can never drift apart.
Since the two new checks never `fail`, `deepReport.summary.failed` is
provably always `runDoctorChecks(resolution).summary.failed` — `--deep` can
only add `pass`/`warn`/`skip`, confirmed by dedicated tests.

### CLI wiring

`apps/cli/src/index.ts`'s `doctor` command gained `--deep` (boolean flag)
and its `.action` became `async`. Without `--deep`: `runDoctorChecks(...)`
(unchanged). With `--deep`: `await runDoctorChecksDeep(...)`. `--json`/exit
code logic is shared by both branches, unchanged.

### Launcher wiring (opted in)

- `RouteMatch`'s `projects.launcherDoctor` variant gained a `deep: boolean`
  field; `matchRoute` parses `?deep=1`/`?deep=true` from the query string
  (same `extractQueryParam` helper `ado-suggestions?type=` already uses).
- `apiGetLauncherDoctor` (`app-launcher.ts`) became `async` and accepts an
  optional `{deep?: boolean}` third parameter; `deep === true` → `await
  runDoctorChecksDeep(resolution)`; otherwise (default) → `runDoctorChecks
  (resolution)` (unchanged fast path).
- `app-api.ts`'s `projects.launcherDoctor` case now `await`s the call and
  forwards `route.deep`.

## Validation

Ran from `C:\repos\Axiom Workspace\Axiom`:

- `npm run build` (`tsc -b`) — PASSED, clean, exit 0.
- `npx vitest run packages/doctor packages/toolchain` — **22 files / 227
  tests passed** (21 pre-existing doctor files unchanged + 1 new
  `deep-checks.test.ts`, 16 tests, covering both new checks' pass/warn/skip
  paths, "never fail" scenarios, and `runDoctorChecksDeep`'s merge +
  `summary.failed` invariant; 3 pre-existing toolchain files unchanged).
  Confirmed the pre-existing `runDoctorChecks` sync `failed` count is
  unaffected by these additions (asserted directly in
  `deep-checks.test.ts`'s `runDoctorChecksDeep` tests: `deepReport.summary
  .failed === syncReport.summary.failed` in both an all-pass and an
  all-warn scenario).
- `apps/cli/tests` files referencing "doctor" (`grep -rln "doctor"
  apps/cli/tests`): `bootstrap.test.ts`, `launcher-doctor.test.ts`,
  `model.test.ts`, `repair.test.ts`, `upgrade-fanout.test.ts` — **5 files /
  52 tests passed** (1 new test added to `launcher-doctor.test.ts` for
  `?deep=1`).
- Extra safety pass: `npx vitest run apps/cli/tests/app.test.ts
  apps/cli/tests/launcher-ado-bridge.test.ts
  apps/cli/tests/launcher-ado-workflow.test.ts
  apps/cli/tests/launcher-execution-mode.test.ts
  apps/cli/tests/launcher-onboarding.test.ts
  apps/cli/tests/launcher-panels.test.ts apps/cli/tests/launcher-push.test.ts`
  (every other file importing `app-api.ts`/`app-launcher.ts`) — **7 files /
  77 tests passed**.
- Full `apps/cli` suite (extra safety pass, since `app-api.ts`/
  `app-launcher.ts`/`index.ts` are widely shared files): **125 files / 1173
  tests passed**, zero failures.

No pre-existing failures were encountered in any touched or adjacent scope.

## Result

Implemented. `axiom doctor --deep` (and the launcher doctor-gate's opt-in
`?deep=1`) now run two new, additive, never-failing runtime probes on top of
the unchanged synchronous check tree:

- `TC-018-<toolId>`: a REAL best-effort spawn/version-check per declared
  tool (reusing `@axiom/toolchain`'s existing `resolveProbeCommand`/
  `probeToolInstalled` contract), honestly `skip`-ping tools with no known
  binary contract (`rtk`/`caveman`) instead of guessing.
- `TC-019-<serverId>`: a REAL MCP `initialize` JSON-RPC handshake against
  the declared launch `command`/`args` for `sdd-mcp-server`/
  `spec-mcp-broker` (read from `.axiom/mcp.yml`), reusing
  `@axiom/providers`'s existing `createStdioMcpClient` transport and
  `@axiom/mcp-server`'s existing `initialize` handler — a genuine "is this
  MCP server alive and speaking the protocol" signal, not just "did a
  process start".

Both are fully injectable for tests (zero real process spawns in the test
suite) and provably incapable of producing `fail`, so `--deep` can only add
`pass`/`warn`/`skip` on top of the exact same `summary.failed` the existing
synchronous gate already computes — the launcher doctor-gate and every
pre-existing doctor test/caller are unaffected unless they explicitly opt
in.

## General spec integration

Not integrated into `Axiom.Spec/specs/00..08` directly — per this task's
brief, do not edit those files. Consistent with the sibling same-day
increments in this repo (`INC-20260726-adapter-mcp-parity` et al.), which
were also left for a later, single integration pass. Stable facts worth
carrying into that future integration pass:

- `axiom doctor` has two modes: the DEFAULT synchronous configuration-only
  check tree (`runDoctorChecks`), and an OPT-IN `--deep` async superset
  (`runDoctorChecksDeep`) that additionally performs REAL runtime probes
  (tool functional probe `TC-018-*`, MCP server liveness `TC-019-*`,
  category `runtime-probes`). Neither new check can ever fail the doctor
  gate; `--deep` only adds `pass`/`warn`/`skip`.
- The MCP liveness probe's source of truth is the project's `.axiom/mcp.yml`
  (`command`/`args` per managed server, written by `axiom workspace
  mcp`/`runWorkspaceSetup`) — a project without one (e.g. self-hosted
  layouts that haven't run workspace setup) gets an honest `warn`, not a
  synthesized guess.
- The launcher doctor-gate endpoint (`GET .../launcher/doctor`) supports the
  same opt-in via `?deep=1`, defaulting to the fast sync path.
