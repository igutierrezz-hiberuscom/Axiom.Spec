# NS-3 — wire code-intel MCP into generated native configs

## Goal

Make the code-intel MCP server (serena/codegraph/graphify) actually appear in the generated
native MCP config of code repos (`.mcp.json` / `.cursor/mcp.json` / `.vscode/mcp.json` /
`opencode.json`), pointed at that repo, **when a code-intel provider is enabled** — without
breaking any project that has none enabled. Axiom already had the provider model
(`@axiom/providers`' serena/codegraph/graphify `ProviderClient`s) and NS-1/NS-2 already
reference "code-intel with physical-path fallback" in the agent prose; NS-3 wires the actual
MCP **server entry** into the generated config so an IDE/adapter can launch it.

## Scope

1. **Emit code-intel MCP server entries** into the generated native MCP configs of **code
   (role) repos**, one per ENABLED code-intel provider, built from the provider client's
   launch command and pointed at that repo's path (serena
   `start-mcp-server --context ide --project <repoPath>`, codegraph
   `serve --mcp -p <repoPath>`, graphify `-m graphify.serve <repoPath>/graphify-out/graph.json`),
   alongside the existing `sdd-mcp-server` / `spec-mcp-broker` entries. Single source of truth:
   `buildCodeIntelNativeServers` in `packages/providers/src/code-intel/native-mcp-launch.ts`,
   shared with the runtime `ProviderClient`s.
2. **Enablement gating (no-regression):** only emit a provider's entry when it is enabled for
   the project — the same code-intel signal the rest of the engine uses
   (`resolveEnabledCodeIntelProviders` in `runWorkspaceSetup`; `workspace.json#providers`
   filtered by `isCodeIntelProviderId` in `runWorkspaceMcpConfig`/`runRepoAdd`). Default empty
   -> nothing emitted -> existing tests + dogfood + single-repo setups byte-identical.
3. **Advertise the fallback** in the generated agent bodies. NS-2 already emits, in both
   `axiom-role-planner` and `axiom-role-implementer`, the "code-intel with physical-path
   fallback" text (degrade to reading the physical repo with Read/Grep when the code-intel MCP
   is unavailable). NS-3 verifies this holds when NO code-intel MCP is wired and adds a
   regression assertion; it does not duplicate the prose.
4. **Wire the emission** into the same paths that build native MCP configs:
   `runWorkspaceSetup` (so `runWorkspaceAdopt`, which delegates to it, is covered),
   `runWorkspaceMcpConfig` (`axiom workspace mcp-config`), and `runRepoAdd` (new code repo).

## Non-goals

- Forcing serena/codegraph on projects that did not opt in (default is empty -> no-op).
- Emitting code-intel into the **spec** repo (planner flow relies on the physical-path
  fallback prose; code repos are the priority — see Decisions).
- Wiring code-intel through `axiom member install` and `axiom adapter add`: both currently
  build native MCP config strictly from `axiom.config/mcp-manifest.yaml` (sdd/spec brokers
  only) and do not emit engram either; member-install's native loop writes only the sdd+spec
  repos, not code repos, so there is no existing code-repo `.mcp.json` emission to extend.
  Deferred consistently with the engram precedent.
- Changing the runtime code-intel `ProviderClient` behavior, the `providers.yaml` fallback
  chain, or any public function signature (all new args are optional/back-compatible).

## Acceptance

- A project that ENABLES serena (+codegraph): the generated code-repo `.mcp.json` (and
  `.cursor/mcp.json` / `.vscode/mcp.json` / `opencode.json`) contains a serena (and codegraph)
  server entry pointed at that repo's path, alongside `sdd-mcp-server` / `spec-mcp-broker`; the
  command matches the provider client's launch shape.
- With NO code-intel provider enabled (default): the generated `.mcp.json` contains ONLY the
  sdd/spec brokers (byte-identical to pre-NS-3) -> existing tests green.
- The generated implementer/planner agent body references the physical-path fallback when no
  code-intel MCP is present.
- Build / test / typecheck / doctor stay green.

## Decisions

- **Code repos only, each pinned to its own path.** serena/codegraph target a code project
  directory; the implementer flow is per code repo. The spec/planner flow relies on the NS-2
  physical-path fallback prose (KVP25's manual `serena-spec` entry is out of scope here).
- **Single source of truth for launch shapes.** `buildCodeIntelNativeServers`
  (`@axiom/providers`) is shared by the runtime clients and the static config emitter, so the
  emitted entry matches the provider client. The static codegraph form adds `-p <repoPath>`
  (the runtime spawns with cwd instead); serena pins via `--project <repoPath>` in both.
- **Native-only, never `.axiom/mcp.yml`.** Like engram, code-intel entries are projected only
  into the per-tool native configs; `.axiom/mcp.yml` stays Axiom's canonical `type:'axiom'`
  broker file.

## Result

Shipped, green on all gates (build / test / typecheck / doctor).

**1. Single source of truth for launch shapes.** New
`packages/providers/src/code-intel/native-mcp-launch.ts` exports `serenaMcpArgs` /
`codegraphMcpArgs` / `graphifyMcpArgs` and `buildCodeIntelNativeServer(s)`. The runtime
serena/codegraph/graphify `ProviderClient`s now build their launch args from the SAME helpers
(byte-identical output — the existing provider tests, which use stub launch specs, stay
green), so the emitted static config matches the provider client by construction. Exported
from the `@axiom/providers` barrel. The static codegraph form pins `-p <repoPath>`
(`pinProject=true`); the runtime keeps `serve --mcp` + `cwd`, documented at both sites.

**2. Emission + enablement gate.** `writeWorkspaceNativeMcpConfigs`
(`apps/cli/src/commands/workspace-mcp.ts`) gained an optional `codeIntelProviders` arg, and
`WorkspaceMcpNativeRepo` gained an optional `kind`. Per repo it now builds `sharedServers`
(sdd/spec brokers + optional engram) and, **only for `kind==='role'` (code) repos when the
enabled set is non-empty**, appends `buildCodeIntelNativeServers(providers, repo.path)` pinned
to that repo. Enablement signal: the code-intel subset (`isCodeIntelProviderId`) of the
project's enabled providers — `resolveEnabledCodeIntelProviders(spec)` in `runWorkspaceSetup`,
and `workspace.json#providers` in `runWorkspaceMcpConfig` / `runRepoAdd` (mirrors how
`engramEnabled` is derived). Empty/omitted, or a non-code repo => nothing emitted, output
byte-identical to pre-NS-3.

**3. Wired paths.** `runWorkspaceSetup` (so `runWorkspaceAdopt`, which delegates to it),
`runWorkspaceMcpConfig` (`axiom workspace mcp-config`), and `runRepoAdd` (new code repo) all
pass `kind` + the enabled code-intel set. `axiom member install` / `adapter add` are
deferred (Non-goals): both build native config strictly from `mcp-manifest.yaml` (brokers
only) and don't emit engram either; member-install's native loop writes only sdd+spec repos,
not code repos.

**4. Fallback advertisement (verified, not duplicated).** NS-2's
`workspace-process-surfaces.ts` already emits, unconditionally in both `axiom-role-planner`
and `axiom-role-implementer`, "code-intel con fallback … **Si el MCP de code-intel no está
disponible, degrada leyendo el repo físico … (read/search/glob)**". Verified end-to-end: with
no provider enabled, the code repo's `.axiom/agents/axiom-role-implementer.md` still contains
that fallback line ("degrada leyendo el repo físico en `.`** (read/search/glob)").

**Tests.** New `packages/providers/tests/code-intel-native-launch.test.ts` (11) covers the arg
builders + ordering/dedup/unknown-id skipping. New blocks in
`apps/cli/tests/workspace-mcp.test.ts` cover per-repo gating (code repo gets
serena+codegraph+brokers pinned to its path; spec/no-kind repos get brokers only), the
no-regression default, engram-id filtering, and an e2e via `runWorkspaceSetup` (enabled vs
disabled) plus the fallback-prose assertion.

**e2e (scratch, built dist).** `runWorkspaceSetup` on a Kvp-shaped multi-repo (Kvp.Sdd /
Kvp.Spec / Kvp.Api):
- serena+codegraph enabled → `Kvp.Api/.mcp.json` = `sdd-mcp-server` + `spec-mcp-broker` +
  `codegraph` (`serve --mcp -p <Kvp.Api>`) + `serena`
  (`start-mcp-server --context ide --project <Kvp.Api>`); `Kvp.Spec/.mcp.json` = brokers only;
- none enabled → `Kvp.Api/.mcp.json` = brokers only. Matches KVP25's real `.mcp.json` shape
  (the owner's manual `--context ide-assistant` differs only in the context token; the shape
  and pinning match).

**Gate:** `npm run build` ✓ · `npm test` 2779 passed / 274 files (0 failures; +16 new over the
2763 baseline; no load-timeout flake this run) · `npm run typecheck` ✓ · `doctor` PASS
(45/57 OK, 0 FALLO, 3 ADVERTENCIA, 9 OMITIDO — unchanged from NS-2).

**Deferred (out of scope):** code-intel via `axiom member install` / `adapter add` (native
config built from `mcp-manifest.yaml`; consistent with engram's absence there); code-intel in
the spec repo for the planner flow (relies on the physical-path fallback prose).
