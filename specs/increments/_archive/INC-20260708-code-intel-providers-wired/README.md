# Increment: Code-intel providers wired (AB4)

Status: closed
Date: 2026-07-08

## Goal

Wire the three code-intel `ProviderClient`s (`codegraph`, `serena`,
`graphify`) on top of the provider execution seam (`@axiom/providers`,
`INC-20260708-provider-runtime-execution-seam`) so
`invokeCapabilityLive('code.knowledgeGraph' | 'code.semanticNavigation' |
'code.structureAnalysis', ...)` actually launches the corresponding LOCAL
tool's MCP server and returns real results, with graceful `not-installed`
degradation when a tool's binary is absent — LOCAL-ONLY, LOCAL DB, no
bundling, no new npm dependencies.

## Context

- `@axiom/providers` (AB3, closed) already provides: `ProviderClient`,
  `ProviderRegistry`, `invokeCapabilityLive` (resolves via
  `@axiom/tool-routing`'s `routeTool`, walks the fallback chain, never
  throws), `createStdioMcpClient` (local stdio JSON-RPC 2.0 spawn + timeout
  + clean shutdown), the `LOCAL_ONLY`/`isLocalTarget` guard, and a reference
  in-process `filesystem` client.
- `@axiom/capability-model`'s `axiom.config/providers.yaml` (the actual
  source of truth, not just `CANONICAL_PROVIDER_IDS`) declares the
  capability mapping this increment implements verbatim:
  - `codegraph` → `code.knowledgeGraph` (fallback: `serena`).
  - `serena` → `code.semanticNavigation` (fallback: `filesystem`).
  - `graphify` → `code.structureAnalysis` (fallback: `codegraph`).
  - No provider in `providers.yaml` maps to `code.impactAnalysis`,
    `code.symbolSearch`, or `code.referenceSearch` today — confirmed by
    reading `providers.yaml` and grepping the codebase; those three
    capability ids remain unmapped and are out of scope here (a future
    increment's concern if/when `providers.yaml` is extended).
- The three external tools were read directly from `C:\repos\codegraph`,
  `C:\repos\serena`, `C:\repos\graphify` (READMEs + source) to confirm exact
  launch commands and MCP tool names — see Implementation notes.
- `apps/cli/src/commands/workspace-setup.ts`'s `runWorkspaceSetup` is the
  multi-repo scaffolding engine; `@axiom/document-bootstrap`'s
  `writeCanonicalAgentsMd`/`renderCanonicalAgentsMd` renders the canonical,
  repo-root `AGENTS.md`.
- No project-level "which code-intel providers are enabled" config/selection
  UX exists yet (reserved for a future increment, referred to as AB6 in the
  batch). This increment therefore threads an explicit, opt-in
  `enabledCodeIntelProviders?: readonly string[]` field through
  `WorkspaceSetupSpec` and `CanonicalAgentsMdIdentity` rather than inventing
  a persisted selection mechanism — omitted/empty is a strict no-op.

## Scope

- `packages/providers/src/code-intel/shared.ts` — `createCodeIntelProviderClient`,
  a shared factory every one of the three clients is built from: spawns a
  local MCP server per `invoke()` call via `createStdioMcpClient`, calls the
  one mapped tool, normalizes `McpToolCallResult` → `ProviderInvokeResult`,
  and classifies spawn/handshake failures into `not-installed` vs. ordinary
  `error` degraded reasons.
- `packages/providers/src/code-intel/{codegraph,serena,graphify}-client.ts` —
  one `ProviderClient` factory per tool (`createCodegraphProviderClient`,
  `createSerenaProviderClient`, `createGraphifyProviderClient`), each
  documenting its exact launch command + tool mapping inline (see
  Implementation notes) and each exposing minimal override options
  (command/context/python-command/graph-path/timeout) for tests and local
  installs — never raw `args`, so a real launch always uses the tool's
  documented, correct arguments.
- `packages/providers/src/code-intel/register.ts` — `registerCodeIntelProviders(registry, opts?)`,
  registers all three in priority order (codegraph → serena → graphify).
- `packages/providers/src/index.ts` — barrel exports for all of the above.
- `apps/cli/src/commands/workspace-code-intel.ts` — `initializeCodeIntelIndexes`,
  a best-effort, opt-in (`enabledCodeIntelProviders`, default `[]` = no-op)
  hook that attempts each enabled provider's local index-build step
  (`codegraph init`, `graphify extract .`; `serena` has no separate init
  step) for one repo. Never throws; degrades to a warning string per
  provider on failure.
- `apps/cli/src/commands/workspace-setup.ts` — new optional
  `WorkspaceSetupSpec.enabledCodeIntelProviders` field; a new best-effort
  step (7c) that calls `initializeCodeIntelIndexes` for each newly-created
  role (code) repo, gated on the list being non-empty; threads the same list
  into `writeCanonicalAgentsMd`'s identity for role repos only.
- `packages/document-bootstrap/src/canonical-agents-md.ts` — new optional
  `CanonicalAgentsMdIdentity.enabledCodeIntelProviders` field and a "Code
  intelligence tools" section (one short guidance line per enabled
  provider, priority-ordered), emitted only when the list is non-empty.
- Hermetic tests: `packages/providers/tests/fixtures/stub-code-intel-mcp-server.mjs`
  (emulates `codegraph_explore`/`find_symbol`/`query_graph`),
  `packages/providers/tests/code-intel-clients.test.ts`,
  `apps/cli/tests/workspace-code-intel.test.ts`, new scenario (j) in
  `apps/cli/tests/workspace-setup.test.ts`, and new "Code intelligence
  tools" scenarios in `packages/document-bootstrap/tests/canonical-agents-md.test.ts`.

## Non-goals

- No project-level provider-selection UX/wizard (reserved for a future
  increment, AB6 in the batch numbering) — `enabledCodeIntelProviders` is a
  plain opt-in list threaded through by whichever caller already knows it,
  not a persisted/resolved config concept yet.
- No provider mapping for `code.impactAnalysis`/`code.symbolSearch`/
  `code.referenceSearch` — `providers.yaml` does not map any provider to
  these capability ids today; extending `providers.yaml` itself is out of
  scope for this increment (it consumes the capability model as-is, same
  guardrail as AB3).
- No changes to `@axiom/tool-routing`, `@axiom/capability-model`, or
  `@axiom/providers`'s existing seam (`invoke.ts`, `registry.ts`,
  `stdio-mcp-client.ts`, `local-only.ts`) — reused as-is.
- No bundling/vendoring of `codegraph`/`serena`/`graphify`, no new npm
  dependencies. Local install of each tool remains the user's
  responsibility; this increment only wires Axiom to call an already-locally-installed
  tool, or degrade cleanly when it is absent.
- No pooled/long-lived MCP server processes — each `invoke()` call spawns,
  calls one tool, and shuts the process down. A pooled/session-scoped
  variant is a possible future optimization, not required by the
  `ProviderClient` contract (`invoke()` has no lifecycle hooks beyond
  itself).
- No integration into `00-08` (reserved for the orchestrator's final
  consolidation pass across the batch) and no git commits.

## Acceptance criteria

- [x] `codegraph`/`serena`/`graphify` `ProviderClient`s exist under
      `packages/providers/src/code-intel/`, each `supports()`-scoped to
      exactly the capability id(s) `providers.yaml` maps to that provider.
- [x] Each client's `invoke()` uses `createStdioMcpClient` (guarded by the
      existing `LOCAL_ONLY`/`isLocalTarget` check inside it) to spawn the
      tool's documented LOCAL MCP launch command, calls the one mapped real
      tool name, and normalizes the result.
- [x] `registerCodeIntelProviders(registry, opts?)` registers all three,
      with `opts` able to override each client's launch command/paths for
      tests and local installs.
- [x] `runWorkspaceSetup` gains an opt-in, best-effort init-on-setup hook
      (`initializeCodeIntelIndexes`) that never fails setup and is a strict
      no-op unless a project explicitly enables at least one code-intel
      provider.
- [x] The canonical `AGENTS.md` gains a short, conditional "Code
      intelligence tools" section describing only the capabilities actually
      wired, emitted only for code repos with at least one enabled
      provider.
- [x] Hermetic tests cover the ok path (stub local MCP server), the
      `capability-unsupported` path, and the `not-installed` path
      (nonexistent local command) for all three clients, plus
      `registerCodeIntelProviders`, the workspace-setup hook, and the
      AGENTS.md guidance block — none depend on the real
      `codegraph`/`serena`/`graphify` binaries being installed.
- [x] `npm run build` (`tsc -b`) is clean.
- [x] `npx vitest run packages/providers apps/cli packages/document-bootstrap`
      passes; `npx vitest run` (full suite) confirms 0 regressions vs. the
      AB3 baseline (`1888/1888`).

## Open questions

None blocking. Ambiguities resolved under Assumptions.

## Assumptions

- **Serena's `--context` value**: the README's own client docs
  (`docs/02-usage/030_clients.md`) recommend `--context ide` for "most
  other clients" (i.e. non-Claude-Code terminal/IDE integrations), which
  fits Axiom (a harness-agnostic capability router) better than
  `--context claude-code`. Used `ide` as the default, overridable via
  `SerenaClientOptions`.
- **Serena tool choice**: `find_symbol` (from `src/serena/tools/symbol_tools.py`'s
  `FindSymbolTool`, confirmed via Serena's PascalCase-Tool →
  snake_case-tool-name convention) was chosen for `code.semanticNavigation`
  as the general-purpose "locate a symbol" entry point; `find_referencing_symbols`
  and `get_symbols_overview` exist but are not wired to a separate
  capability id (none maps to them in `providers.yaml` today).
- **CodeGraph tool choice**: `codegraph_explore` is CodeGraph's *only*
  listed MCP tool by design (README §MCP Tools: "one strong tool steers
  agents better than a menu of narrower ones") — no ambiguity here.
- **Graphify tool choice**: `query_graph` (BFS/DFS graph search) was chosen
  for `code.structureAnalysis` over the more specific `get_node`/
  `get_neighbors`/`shortest_path` because its own description ("Search the
  knowledge graph... returns relevant nodes and edges as text context")
  most directly matches "structure analysis" / surveying an area, per the
  brief's own hint ("structure analysis").
- **Graphify graph-path resolution**: defaults to
  `<projectRoot>/graphify-out/graph.json` (the path Graphify's own README
  documents as the default build output) and is overridable via
  `GraphifyClientOptions.resolveGraphPath` for tests/non-standard layouts.
- **`not-installed` classification**: `StdioMcpClientError` kinds
  `spawn-failed` (rejected synchronously by the `LOCAL_ONLY` guard or a
  thrown `spawn()`) and `closed` (the child process exited before ever
  answering `initialize` — covers the very common real-world case of a
  missing binary surfacing as `ENOENT` via the child's `error` event,
  confirmed empirically on Windows) both map to `not-installed`;
  `timeout`/`protocol-error` map to ordinary `error` (the process DID
  start, so installation is not the issue).
- **Per-invocation subprocess lifetime**: each `ProviderClient.invoke()`
  call spawns a fresh MCP server process and tears it down in a `finally`
  block after one `tools/call`. This is the simplest correct behavior
  matching the seam's per-call contract; it is NOT how a human would run
  these tools interactively (where a long-lived server serving many calls
  across a session is the norm), but pooling/reuse is an orthogonal future
  optimization the `ProviderClient` interface does not require.
- **`enabledCodeIntelProviders` as a plain opt-in list** (not a resolved
  `ResolvedInstallProfile`/`providers.yaml`-driven selection): the brief
  explicitly says "Gate behind the providers the project actually
  selected (AB6 adds selection; for now, make it opt-in/no-op unless a
  provider is configured" — interpreted as: thread a simple, caller-supplied
  list through both `WorkspaceSetupSpec` and `CanonicalAgentsMdIdentity`,
  defaulting to empty/no-op everywhere, rather than inventing a persisted
  selection mechanism that AB6 would likely replace anyway.
- **AGENTS.md guidance scoped to code (`kind: 'role'`) repos only**: the
  control/spec repos never invoke `code.*` capabilities themselves, so the
  "Code intelligence tools" section is only threaded through for `kind:
  'role'` repos in `workspace-setup.ts`'s `writeOneRepo`, even though
  `renderCanonicalAgentsMd` itself is repo-kind-agnostic (any caller could
  pass the field for any repo).
- **Windows `cwd` gotcha (test-authoring finding, not a product bug)**: on
  Windows, `child_process.spawn` fails with `ENOENT` for the *executable
  itself* (not a directory-not-found error) when the requested `cwd` does
  not exist — discovered while first writing the hermetic stub-server
  tests with a fake `/tmp/...` `projectRoot`, which made every "ok path"
  test look like a false `not-installed` degrade. Fixed by using a real
  directory (`os.tmpdir()`) as the test `projectRoot`; documented inline in
  `code-intel-clients.test.ts` so it is not rediscovered by a future editor
  of these tests. This does not affect real usage, since a real
  `ProviderInvokeContext.projectRoot` is always an existing project
  directory.

## Implementation notes

**Per-tool launch command + MCP tool names (confirmed from each tool's own
README/source under `C:\repos\<tool>`):**

| Provider | Local binary/launch command | MCP tool used | Capability wired |
|---|---|---|---|
| `codegraph` | `codegraph serve --mcp` (stdio; `bin` field in `codegraph`'s `package.json` maps `codegraph` → its bundled runtime) | `codegraph_explore` (CodeGraph's *only* listed MCP tool) | `code.knowledgeGraph` |
| `serena` | `serena start-mcp-server --context ide --project <projectRoot>` (installed via `uv tool install -p 3.13 serena-agent`) | `find_symbol` (from `FindSymbolTool` in `src/serena/tools/symbol_tools.py`) | `code.semanticNavigation` |
| `graphify` | `python -m graphify.serve <projectRoot>/graphify-out/graph.json` (installed via `uv tool install graphifyy`/`graphifyy[mcp]`; default transport is stdio) | `query_graph` (from `graphify/serve.py`'s `list_tools()`) | `code.structureAnalysis` |

Files created:
- `Axiom/packages/providers/src/code-intel/shared.ts` — `createCodeIntelProviderClient`,
  `classifyLaunchFailure`, `normalizeToolCallResult`.
- `Axiom/packages/providers/src/code-intel/codegraph-client.ts` —
  `createCodegraphProviderClient`.
- `Axiom/packages/providers/src/code-intel/serena-client.ts` —
  `createSerenaProviderClient`.
- `Axiom/packages/providers/src/code-intel/graphify-client.ts` —
  `createGraphifyProviderClient`.
- `Axiom/packages/providers/src/code-intel/register.ts` —
  `registerCodeIntelProviders`.
- `Axiom/packages/providers/tests/fixtures/stub-code-intel-mcp-server.mjs` —
  hermetic stub emulating `codegraph_explore`/`find_symbol`/`query_graph`.
- `Axiom/packages/providers/tests/code-intel-clients.test.ts`.
- `Axiom/apps/cli/src/commands/workspace-code-intel.ts` —
  `initializeCodeIntelIndexes`, `runBestEffort` (exported for direct testing).
- `Axiom/apps/cli/tests/workspace-code-intel.test.ts`.

Files edited:
- `Axiom/packages/providers/src/index.ts` — barrel exports for the new
  code-intel module.
- `Axiom/apps/cli/src/commands/workspace-setup.ts` — new
  `WorkspaceSetupSpec.enabledCodeIntelProviders` field, new best-effort
  step 7c (`initializeCodeIntelIndexes` per newly-created role repo, gated
  on the list being non-empty), threads the same list into
  `writeCanonicalAgentsMd`'s identity for `kind: 'role'` repos only.
- `Axiom/apps/cli/tests/workspace-setup.test.ts` — new Scenario (j): no-op
  when omitted/empty, best-effort warning-not-failure when a provider is
  enabled without local wiring (`serena`, which has no init step, used to
  keep the test hermetic regardless of what happens to already be
  installed on the machine running the tests — this machine, for instance,
  already has a real `codegraph` on `PATH`).
- `Axiom/packages/document-bootstrap/src/canonical-agents-md.ts` — new
  optional `CanonicalAgentsMdIdentity.enabledCodeIntelProviders` field,
  `renderCodeIntelGuidanceBlock`, wired into `renderCanonicalAgentsMd`.
- `Axiom/packages/document-bootstrap/tests/canonical-agents-md.test.ts` —
  new "Code intelligence tools" guidance scenarios.

No changes to `@axiom/tool-routing`, `@axiom/capability-model`, or any file
under `Axiom.Spec/specs/00-08`.

**How `not-installed` degrades**: every client returns
`{ ok: false, degraded: true, reason: 'not-installed', message:
"Provider '<id>' is not installed locally (command: \"<command>\"). Install
it and re-run, or configure a different provider. Detail: <spawn detail>"
}` — actionable (names the exact command that failed) without ever
throwing. `initializeCodeIntelIndexes` mirrors this with a Spanish-language
warning string (matching `workspace-setup.ts`'s existing warning-message
convention) naming the tool and linking to its install docs, and NEVER
causes `runWorkspaceSetup` to fail or skip any other step — confirmed by
Scenario (j)'s tests, which assert `result.repos[...].created === true` and
a full `axiom.yaml` scaffold even when a code-intel provider is enabled and
unavailable.

## Validation

- `npm run build` (`tsc -b`, from `Axiom/`): clean, no errors.
- `npx vitest run packages/providers apps/cli packages/document-bootstrap`:
  ```
  Test Files  68 passed (68)
       Tests  581 passed (581)
  ```
- `npx vitest run` (full suite, parallel — default config):
  ```
  Test Files  184 passed | 1 failed (185)
       Tests  1916 passed | 1 failed (1917)
  ```
  The one failure
  (`packages/providers/tests/stdio-mcp-client.test.ts > times out a call
  that the server never answers`) is a PRE-EXISTING AB3 test with a
  hard-coded 200ms timeout, unrelated to this increment's files; it is
  CPU-load-sensitive under full parallel execution. Confirmed pre-existing
  and environment-only by (a) re-running that file alone — passes 6/6 —
  and (b) re-running the ENTIRE suite with `--no-file-parallelism`:
  ```
  Test Files  185 passed (185)
       Tests  1917 passed (1917)
  ```
  `1917/1917` passing, 0 failures. Baseline (AB3 closing state) was
  `1888/1888`; delta is `+29` new tests (`+1` test file:
  `code-intel-clients.test.ts` with 12; `workspace-code-intel.test.ts` with
  9 new in a new file; `+3` in `workspace-setup.test.ts` Scenario (j); `+5`
  in `canonical-agents-md.test.ts`'s new guidance scenarios), all new, all
  passing — 0 regressions.
- `npm run doctor` (self-dogfood, extra sanity): `41/54 OK · 0 FALLO · 3
  ADVERTENCIA · 10 OMITIDO · Resultado: PASS` — identical to AB3's closing
  state; nothing new introduced by this increment.

## Result

The three code-intel providers are wired end-to-end on the AB3 seam:
`createCodegraphProviderClient`/`createSerenaProviderClient`/
`createGraphifyProviderClient` (all built from a shared
`createCodeIntelProviderClient` factory) each spawn their tool's documented
LOCAL MCP launch command via the existing `createStdioMcpClient`, call the
one real, confirmed MCP tool name per provider, and normalize the result —
`registerCodeIntelProviders` registers all three in priority order
(codegraph → serena → graphify, matching `providers.yaml`'s own fallback
chain). `runWorkspaceSetup` gained a strictly opt-in, best-effort
`initializeCodeIntelIndexes` hook (never runs unless a project explicitly
enables at least one provider, never fails setup), and the canonical
`AGENTS.md` gained a short, conditional "Code intelligence tools" guidance
section describing only what is actually wired. One real bug was found and
fixed during test-authoring (not a product bug): a fake `/tmp/...`
`projectRoot` in tests caused Windows `spawn` to fail with a misleading
`ENOENT` on the executable itself when `cwd` did not exist — fixed by using
`os.tmpdir()`, documented inline for future editors. Full suite went from
`1888/1888` to `1917/1917` passing (`+29` new tests, `0` regressions,
confirmed via a non-parallel full run after one CPU-load-induced flake in a
pre-existing, unrelated AB3 test).

## General spec integration

Integrated into `Axiom.Spec/specs/00-08` by the batch's final
consolidation pass (autopilot orchestrator). Landed in:

- **06_Integraciones_y_Capacidades.md** — PRIMARY. New subsection
  "Providers de code-intel cableados (codegraph/serena/graphify)" with
  the per-tool launch-command/MCP-tool/capability table; contributes to
  superseding the "declared-not-wired" framing for these three
  providers. The still-unmapped
  `code.impactAnalysis`/`code.symbolSearch`/`code.referenceSearch` gap
  is noted as an honest, deferred hole.
- **01_Requisitos_Funcionales.md** — RF-AXM-025 cites this increment for
  the concrete-clients half.
- **08_Glosario.md** — covered under the `@axiom/providers` /
  `invokeCapabilityLive` terms.

Reconciled with AB6 (`wizard-configure-provider-selection`), which
superseded this increment's plain `enabledCodeIntelProviders` opt-in
list with the persisted `workspace.json#providers` selection.
