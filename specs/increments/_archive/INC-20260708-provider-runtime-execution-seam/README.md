# Increment: Provider runtime execution seam (AB3)

Status: closed
Date: 2026-07-08

## Goal

Build the missing EXECUTION seam so capability→provider resolutions
produced by `@axiom/tool-routing`'s dispatcher can actually be invoked at
runtime, local-only, with local DBs. Today the dispatcher is a pure
mapper: it resolves `providerEffective` (+ fallback chain) but nothing
executes the provider — execution is "delegated to callers" that do not
exist. This is the root of the "declared-not-wired" debt for
`filesystem`, `serena`, `codegraph`, `graphify`, `engram`,
`axiom-gateway`. This increment builds the seam only; concrete
MCP-backed provider clients (serena/codegraph/graphify in AB4,
engram/memory in AB5) are out of scope and build on top of this seam.

## Context

- `@axiom/capability-model` declares 7 canonical provider ids
  (`filesystem`, `axiom-gateway`, `serena`, `codegraph`, `graphify`,
  `engram`, `generated-snapshots`) with `ProviderDefinition` metadata
  (`kind`, `applicableCapabilities`, `supportsDiscoveryModes`, etc.) but
  no execution behavior — it is pure data.
- `@axiom/tool-routing`'s `routeTool` (in `dispatcher.ts`) is explicitly
  documented as a PURE function: "does NOT perform filesystem reads, MCP
  connections, or network I/O." It returns a `ResolvedDispatch` with
  `providerEffective` and `fallbacksUsed`, and the module comment says
  execution is delegated to callers — no such caller exists anywhere in
  the codebase (confirmed by reading `dispatcher.ts`, `select.ts`,
  `fallback.ts`, `types.ts`, `index.ts` in full).
  `@axiom/mcp-server` (`stdio.ts`, `server.ts`, `protocol.ts`) already
  has a working hand-rolled JSON-RPC-2.0-over-newline-delimited-stdio
  implementation, used today only for Axiom's own `sdd`/`spec` MCP
  *servers* (the listening side). No equivalent *client* transport exists
  to speak to an external local MCP-backed tool (serena, codegraph,
  graphify, engram) as a subprocess.
- User constraint (explicit, non-negotiable): "todas las herramientas
  siempre en local con bbdd en local" — all tools always run locally
  with local databases. No cloud/remote provider execution paths are
  permitted anywhere in this seam.

## Scope

- New package `@axiom/providers` (mirrors `@axiom/tool-routing`'s
  package.json/tsconfig shape; no new npm dependencies).
- `ProviderClient` interface + `ProviderInvokeResult` discriminated
  union (ok/degraded, never throws).
- `ProviderRegistry`: register/lookup a `ProviderClient` by provider id.
- `invokeCapabilityLive(capabilityId, input, ctx, { registry, routing })`:
  resolves the effective provider + fallback chain via the EXISTING
  `@axiom/tool-routing` dispatcher (`routeTool`), then executes through
  the registered `ProviderClient`, walking the fallback chain on
  degrade. Never throws.
- `createStdioMcpClient({ command, args, cwd, env })`: reusable LOCAL
  stdio MCP client helper that spawns a local child process, speaks
  newline-delimited JSON-RPC 2.0 (`initialize` / `tools/list` /
  `tools/call`), and cleanly shuts down. Timeouts + child-process
  cleanup included. This is the pattern AB4/AB5's concrete clients will
  reuse; no concrete client is implemented here beyond one trivial
  reference.
- `LOCAL_ONLY` guard/constant: rejects/degrades any provider config
  pointing at a non-local endpoint.
- One trivial in-process reference `ProviderClient` (`filesystem`) to
  prove the seam end-to-end and serve as the implementation pattern for
  AB4/AB5.
- Root `tsconfig.json` project references + `vitest.config.ts` alias
  wiring for the new package.

## Non-goals

- No concrete `serena`/`codegraph`/`graphify` MCP clients (AB4).
- No concrete `engram`/memory client (AB5).
- No CLI surface, no wizard/configure provider selection (AB6).
- No new npm dependencies (no `@modelcontextprotocol/sdk`).
- No changes to `@axiom/tool-routing`'s dispatcher contract — reused
  as-is.
- No remote/cloud provider execution path of any kind.
- No integration into `Axiom.Spec/specs/00-08` (reserved for the
  orchestrator's final consolidation pass across the batch).
- No git commits.

## Acceptance criteria

- [x] `@axiom/providers` package exists, builds under `tsc -b` as part
      of the root project-reference graph.
- [x] `ProviderClient`, `ProviderInvokeResult`, `ProviderRegistry` are
      exported with the exact shapes specified in the brief.
- [x] `invokeCapabilityLive` resolves via the existing
      `@axiom/tool-routing` dispatcher (no reimplementation of routing),
      executes the registered client, walks the fallback chain on
      degrade, and never throws — missing/uninstalled providers yield a
      typed degraded result.
- [x] `createStdioMcpClient` spawns a local process, performs
      `initialize` → `tools/list` → `tools/call`, shuts down cleanly,
      and enforces timeouts.
- [x] `LOCAL_ONLY` guard rejects any non-local endpoint/URL.
- [x] One reference `ProviderClient` proves the seam end-to-end.
- [x] `packages/providers/tests/*` covers registry, `invokeCapabilityLive`
      routing/fallback/degrade, the stdio client round-trip against a
      stub local server, and the `LOCAL_ONLY` guard.
- [x] Full suite stays green at the pre-existing baseline (no
      regressions).

## Open questions

None blocking. Ambiguities resolved under Assumptions.

## Assumptions

- "Mirror an existing leaf package" is interpreted as
  `@axiom/tool-routing`'s package.json/tsconfig shape (same monorepo
  conventions: `composite: true`, `outDir: dist`, `rootDir: src`,
  `paths`/`references` for its one workspace dependency).
- The stdio MCP *client* is a new, separate module from
  `@axiom/mcp-server`'s stdio *server* transport (`runStdioServer`
  dispatches inbound requests to a local `McpServer`; a client instead
  writes outbound requests to a spawned child's stdin and reads
  responses from its stdout). "Reuse its transport approach" is
  interpreted as reusing the newline-delimited JSON-RPC 2.0 framing
  convention and request/response shapes (`protocol.ts`), not literally
  importing server-side code into the client, since the two sides read
  the opposite ends of the pipe. No new dependency on `@axiom/mcp-server`
  was added; the JSON-RPC types needed by the client are declared locally
  in `@axiom/providers` (small, local subset) to avoid a
  provider→mcp-server workspace dependency that would not otherwise
  exist.
- "Stub local server" for the stdio-client test is a tiny inline Node.js
  script (spawned via `node -e ...` / a temp `.mjs` file) that speaks
  the same newline-delimited JSON-RPC 2.0 framing and implements
  `initialize`/`tools/list`/`tools/call`, rather than wiring
  `@axiom/mcp-server`'s real server through a pipe — this keeps
  `@axiom/providers` free of a workspace dependency on `@axiom/mcp-server`
  and keeps the test hermetic (no reliance on another package's
  behavior beyond the wire protocol both already agree on).
- The one "trivial reference client" is an in-process `filesystem`
  `ProviderClient` (not a real MCP subprocess) — filesystem access needs
  no external local server, and this keeps the reference minimal while
  still exercising `ProviderRegistry` + `invokeCapabilityLive` end to
  end, matching the brief's "e.g. an in-process filesystem provider
  client" suggestion.
- `LOCAL_ONLY` rejection is implemented as a pure string-based host
  check (`localhost`, `127.0.0.1`, `::1`, or no scheme at all i.e. a
  bare local command) — no DNS resolution is performed (would violate
  "local-only, no network I/O" for the guard itself).
- Baseline full-suite count referenced by the brief (1859/1859) is
  taken as the AB1+AB2 closed-state baseline; this increment's own
  validation re-confirms the actual current count rather than assuming
  it blindly, since AB3 runs after AB1/AB2 in the same batch.

## Implementation notes

Files created:
- `Axiom/packages/providers/package.json`
- `Axiom/packages/providers/tsconfig.json`
- `Axiom/packages/providers/src/types.ts` — `ProviderClient`,
  `ProviderInvokeResult`, `ProviderInvokeContext`, `DegradedReason`.
- `Axiom/packages/providers/src/registry.ts` — `ProviderRegistry`
  (`register`/`lookup`/`has`/`ids`).
- `Axiom/packages/providers/src/local-only.ts` — `LOCAL_ONLY` guard
  (`assertLocalTarget`/`isLocalTarget`) + doc comment.
- `Axiom/packages/providers/src/invoke.ts` — `invokeCapabilityLive`.
- `Axiom/packages/providers/src/mcp-client-protocol.ts` — minimal local
  JSON-RPC 2.0 request/response types for the stdio client (mirrors
  `@axiom/mcp-server/src/protocol.ts`'s shapes without importing it).
- `Axiom/packages/providers/src/stdio-mcp-client.ts` —
  `createStdioMcpClient`.
- `Axiom/packages/providers/src/filesystem-client.ts` — reference
  in-process `filesystem` `ProviderClient`.
- `Axiom/packages/providers/src/index.ts` — barrel.
- `Axiom/packages/providers/tests/registry.test.ts`
- `Axiom/packages/providers/tests/invoke-capability-live.test.ts`
- `Axiom/packages/providers/tests/stdio-mcp-client.test.ts`
- `Axiom/packages/providers/tests/local-only.test.ts`
- `Axiom/packages/providers/tests/filesystem-client.test.ts`
- `Axiom/packages/providers/tests/fixtures.ts` — shared `RouteToolContext`
  /profile/capability-model fixture builders (mirrors the pattern in
  `packages/tool-routing/tests/route-tool.test.ts`).
- `Axiom/packages/providers/tests/fixtures/stub-mcp-server.mjs` — stub
  local JSON-RPC server used only by the stdio-client test.

Files edited:
- `Axiom/tsconfig.json` — added `{ "path": "packages/providers" }` to
  root `references`.
- `Axiom/vitest.config.ts` — added `@axiom/providers` alias.

No changes to `@axiom/tool-routing` or `@axiom/capability-model` — both
consumed as-is, per the guardrail against reimplementing routing.

## Validation

- Baseline (before this increment's code changes), `npx vitest run`
  (from `Axiom/`): `1859 passed (1859)`, `178 Test Files passed (178)` —
  matches AB1+AB2's stated closing baseline exactly.
- `npm run build` (from `Axiom/`, `tsc -b`): clean, no errors, new
  package compiles as part of the reference graph.
- `npx vitest run packages/providers packages/tool-routing packages/mcp-server`:
  ```
  Test Files  9 passed (9)
       Tests  97 passed (97)
  ```
  (`packages/providers` contributes 6 test files / 29 tests: registry 5,
  local-only 6, invoke-capability-live 8, filesystem-client 4,
  stdio-mcp-client 6.)
- `npx vitest run` (full suite, after this increment's code changes):
  ```
  Test Files  183 passed (183)
       Tests  1888 passed (1888)
  ```
  Delta vs. baseline: `+5 test files / +29 tests`, all new, all passing;
  `0` failing both before and after — no regressions.
- `npm run doctor` (self-dogfood, extra sanity beyond the requested
  validation): `41/54 OK · 0 FALLO · 3 ADVERTENCIA · 10 OMITIDO` ·
  `Resultado: PASS` — same pre-existing benign warnings/skips as before
  this increment (`TC-012`/`TC-013` optional structures, `WS-001` no
  active plan, `DF-001` single-repo mode), nothing new introduced by
  `@axiom/providers`.

## Result

The provider execution seam is implemented as `@axiom/providers`:
`ProviderClient`/`ProviderRegistry`/`invokeCapabilityLive` (built on the
existing, unmodified `@axiom/tool-routing` dispatcher for resolution),
`createStdioMcpClient` (local-only spawn + newline-delimited JSON-RPC
2.0 + timeout + clean shutdown), the `LOCAL_ONLY` guard, and one
reference in-process `filesystem` `ProviderClient` proving the seam
end-to-end. One test-writing detour was needed and resolved during
implementation: two `invokeCapabilityLive` tests initially asserted a
narrower outcome (`capability-unsupported` / `error`) than what the
shared multi-provider fixture model actually produced, because the
default fixture profile makes both `filesystem` and `codegraph` viable
routing candidates for `code.structureAnalysis` — the fallback walk
correctly continued to the unregistered `codegraph` and reported
`provider-unavailable` instead. Fixed by scoping those two tests'
profile to `filesystem` only, which is the correct fix (the seam's
fallback-walking behavior was right; the test isolation was not) and
is now called out explicitly in each test's comments. Full suite went
from `1859/1859` passing to `1888/1888` passing (`+29` new tests, `0`
regressions). `npm run build` is clean across the updated reference
graph.

## General spec integration

Integrated into `Axiom.Spec/specs/00-08` by the batch's final
consolidation pass (autopilot orchestrator). Landed in:

- **06_Integraciones_y_Capacidades.md** — PRIMARY. New subsection "Seam
  de ejecución de providers (`@axiom/providers`)"; also superseded the
  "declared-not-wired" framing in the "Capability model y providers"
  section (the execution layer now exists).
- **03_Modelo_Operativo_y_Datos.md** — cited in the INC-20260708 data
  subsection as the transport `createEngramBackend`/code-intel clients
  reuse.
- **08_Glosario.md** — new terms `@axiom/providers` / provider
  execution seam and `invokeCapabilityLive`.
- **01_Requisitos_Funcionales.md** — RF-AXM-025 (provider execution +
  selection) cites this increment for the seam half.
