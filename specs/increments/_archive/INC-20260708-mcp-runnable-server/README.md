# Increment: MCP runnable server (sdd-mcp-server / spec-mcp-broker)

Status: closed
Date: 2026-07-08

## Goal

Today `sdd-mcp-server` / `spec-mcp-broker` are only string identifiers
written into `mcp.yml` — there is NO runnable MCP server. Make them REAL:
build a minimal but correct MCP server (JSON-RPC 2.0 over newline-delimited
stdio) that exposes Axiom's existing in-process `@axiom/mcp-tools` handlers,
plus an `axiom mcp serve` CLI command to launch it.

## Context

`@axiom/mcp-tools` already registers 17 stateless tool handlers
(`MCP_TOOL_HANDLERS`, `invokeMcpTool(capabilityId, input)`) split into
`sdd.*` (registry/skills/index/validation operations) and `spec.*`
(read-only artifact/index/catalog reads) domains. `apps/cli/src/commands/
mcp.ts` already has an `axiom mcp` command family (`list`/`validate`/
`repair`/`inventory`) that manages a human-facing `mcp-manifest.yaml`
catalog of *capabilities* — this is deliberately unrelated to the actual
server *processes* that would serve those capabilities over the MCP
protocol (see that file's own header comment on the distinction from
`workspace-mcp.ts`'s `mcp.yml`). No code in the repo implements the MCP
wire protocol (JSON-RPC framing, `initialize`/`tools/list`/`tools/call`)
or a stdio transport loop. This increment adds exactly that runtime, wired
to the existing handlers — it does not change the manifest/catalog or the
`mcp.yml`/adapter-config generators.

`apps/cli/src/commands/mcp.ts` is compiled by `@axiom/cli-commands`
(single-ownership build rule from `INC-20260703-cli-commands-single-
ownership`): the file physically lives under `apps/cli/src/commands/` but
is `include`d in `packages/cli-commands/tsconfig.json` and `exclude`d from
`apps/cli/tsconfig.json`; `apps/cli/src/index.ts` imports `registerMcp`
from `@axiom/cli-commands`, never from a relative path. Any new export from
`mcp.ts` must follow the same seam.

## Scope

- New package `@axiom/mcp-server` (`packages/mcp-server/`):
  - `createMcpServer({ serverKind, context })`: transport-agnostic, pure,
    testable JSON-RPC 2.0 request/response dispatcher.
  - `runStdioServer(server, { stdin, stdout })`: newline-delimited JSON-RPC
    stdio transport loop.
  - `resolveMcpServerContext({ projectRoot, homeDir? })`: resolves
    `McpServerContext` (home dir, project root, project id, spec scope
    path, role) from the existing `@axiom/project-resolution`,
    `@axiom/user-workspace`, `@axiom/topology` primitives.
  - Per-capability input-builders covering every capability in each
    server's tool set.
- CLI: `runMcpServe` + `axiom mcp serve --kind <sdd|spec> [--project-root
  <path>] [--home-dir <path>]` in a NEW app-owned file,
  `apps/cli/src/commands/mcp-serve.ts` (`registerMcpServe`, called from
  `apps/cli/src/index.ts` right after `registerMcp`, reusing the same
  `mcp` commander sub-command group). See "Implementation notes" for why
  this lives in a separate file rather than inline in the existing
  `mcp.ts` (a real project-reference cycle, discovered while wiring the
  build).
- Focused tests: `packages/mcp-server/tests/{server,stdio}.test.ts`,
  `apps/cli/tests/mcp-serve.test.ts`.
- Root `tsconfig.json` gains a `{ "path": "packages/mcp-server" }`
  reference; `vitest.config.ts` gains an `@axiom/mcp-server` alias;
  `apps/cli/tsconfig.json` gains a `@axiom/mcp-server` path + reference
  (needed by the new `mcp-serve.ts`, which IS compiled by `apps/cli`
  itself, unlike `mcp.ts`).

## Non-goals

- Do NOT change the `mcp.yml` / `mcp-manifest.yaml` shapes, or the adapter
  `mcp.json` generators (`generateOpencodeMcpJson` /
  `generateClaudeCodeMcpJson`). Wiring the `axiom mcp serve` launch command
  into those generated configs is a follow-up increment.
- Do NOT add new npm dependencies. No `@modelcontextprotocol/sdk` — the
  JSON-RPC/stdio transport is hand-rolled, kept minimal but spec-correct
  enough for a standard MCP client to `initialize` + `tools/list` +
  `tools/call`.
- Do NOT implement a real child-process spawn end-to-end test (heavy e2e is
  deferred); tests drive the server/stdio loop via in-memory streams and
  direct function calls only.
- Do NOT touch `00`–`08` canonical spec docs in this increment.

## Acceptance criteria

- [x] `@axiom/mcp-server` exports `createMcpServer`, `runStdioServer`,
      `resolveMcpServerContext`, the sdd/spec tool-set constants, and their
      types from a single barrel (`src/index.ts`).
- [x] `createMcpServer` handles `initialize` (protocolVersion +
      capabilities.tools + serverInfo), `notifications/initialized` (no
      reply), `tools/list` (tool set scoped to `serverKind`), `tools/call`
      (dispatches to `invokeMcpTool` via a per-capability input-builder,
      wraps `ok`/`err` into MCP `content`/`isError`), `ping`, and an
      unknown-method JSON-RPC error (`-32601`).
- [x] `runStdioServer` reads newline-delimited JSON-RPC from an injectable
      stdin-like stream and writes newline-delimited responses to an
      injectable stdout-like stream; tolerates blank lines; replies
      `-32700` on parse errors when an id is recoverable.
- [x] `resolveMcpServerContext` derives `{ homeDir, projectRoot, projectId,
      specScopeAbsolutePath, role }` using `resolveProject`/
      `resolveHomeDir`/`loadTopology`/`resolveRepoPath`/
      `findByAncestorRepoPathV2`, best-effort (never throws on missing
      registry/topology data).
- [x] `axiom mcp serve --kind sdd|spec [--project-root <path>] [--home-dir
      <path>]` is registered (via `registerMcpServe` in the new
      `mcp-serve.ts`, wired from `apps/cli/src/index.ts`) and its pure
      `runMcpServe` function is directly importable for tests.
- [x] `packages/mcp-server/tests/server.test.ts` and `.../stdio.test.ts`,
      plus `apps/cli/tests/mcp-serve.test.ts`, pass.
- [x] `npm run build` (tsc -b) is clean and picks up the new package via
      the root tsconfig references.
- [x] `npx vitest run packages/mcp-server apps/cli packages/mcp-tools`
      passes (pre-existing `packages/skills/tests/catalog.test.ts` dogfood
      failure excluded, out of scope).

## Open questions

None blocking — all resolved under "Assumptions" below per the increment
prompt's explicit instruction to proceed without stopping.

## Assumptions

- **Tool set split (sdd vs spec)**: the `serverKind` maps 1:1 to
  `McpToolRegistryEntry.domain` — `'sdd'` server exposes every capability
  whose id starts with `sdd.` (7 capabilities), `'spec'` server exposes
  every capability whose id starts with `spec.` (10 capabilities,
  including the composite `spec.implementationContextRead`). This mirrors
  `buildWorkspaceMcpServers`' existing `sdd-mcp-server`/`spec-mcp-broker`
  split by role, and the `MCP_TOOL_HANDLERS` registry's own `domain` field
  — no new classification invented.
- **`role` resolution**: `McpServerContext.role` is best-effort and has no
  strong source of truth today (no per-project "current role" concept
  exists outside a plan's `taskType`). Left `undefined` when it cannot be
  inferred; input-builders that need a `role` (`sdd.skillIndexRead`,
  `spec.technicalContextIndexRead`, `spec.recommendedContextList`) accept
  it as a `tools/call` argument override and fail with a clear `isError`
  tool result (not a throw) when neither the context nor the call args
  supply one.
- **`specScopeAbsolutePath` resolution**: resolved via
  `loadTopology(projectRoot)` → `resolveRepoPath(topology.specRepo,
  localBindings, projectRoot)`. Falls back to `projectRoot` itself when
  topology loading fails (single-repo default behavior, matching
  `defaultSingleRepoManifest`'s own convention that `specRepo` coincides
  with `projectRoot`).
- **`projectId` resolution**: resolved via
  `findByAncestorRepoPathV2(homeDir, projectRoot)` (ancestor-match against
  the registered project repos in `~/.axiom/projects.yml`), falling back to
  `undefined` when the project is not registered — capabilities that need
  `projectId` (`sdd.projectReposRead`, `spec.implementationContextRead`)
  surface a clear `isError` when it is missing, mirroring the
  missing-required-arg contract used everywhere else.
- **Missing required args**: per the increment's design decision, a
  missing required input field for a `tools/call` never throws — it
  returns `{ content: [...], isError: true }` with a message naming the
  missing field(s).
- No real adopting workspace launches this server yet (confirmed by
  reading `workspace-mcp.ts`: `mcp.yml` only records `id`/`type`/`scope`/
  `targetRepo`/`enabled`, no launch command), so there is no backward
  compatibility surface to preserve for this increment.

## Implementation notes

### Package layout

- `packages/mcp-server/src/protocol.ts`: JSON-RPC 2.0 types + error codes
  + minimal MCP shapes (`initialize` result, tool descriptor, tool-call
  result).
- `packages/mcp-server/src/context.ts`: `McpServerContext` +
  `resolveMcpServerContext` (best-effort, never throws).
- `packages/mcp-server/src/tool-sets.ts`: `SDD_TOOL_CAPABILITY_IDS` (7
  ids) / `SPEC_TOOL_CAPABILITY_IDS` (10 ids), both derived directly from
  `MCP_TOOL_HANDLERS`' own `domain` field — not hand-maintained lists.
  Also builds the `tools/list` `McpToolDescriptor[]` (name + one-liner
  description + permissive `{type:'object'}` inputSchema).
- `packages/mcp-server/src/input-builders.ts`: one pure
  `(context, args) => BuildResult` function per capability id (17 total),
  registered in `INPUT_BUILDERS`. Field names verified directly against
  each handler's own input interface in `@axiom/mcp-tools`.
- `packages/mcp-server/src/server.ts`: `createMcpServer` — the pure
  JSON-RPC dispatcher.
- `packages/mcp-server/src/stdio.ts`: `runStdioServer` — the
  newline-delimited stdio transport loop.
- `packages/mcp-server/src/index.ts`: barrel.

### CLI wiring — and the project-reference cycle that shaped it

The original plan was to add `runMcpServe` + the `serve` sub-command
directly inside the existing `apps/cli/src/commands/mcp.ts` (which is
compiled by `@axiom/cli-commands` under the INC-20260703 single-ownership
rule) and re-export it through `@axiom/cli-commands`'s barrel, matching
every other `mcp.ts` export. Attempting this produced a real `tsc -b`
error (`TS6202: Project references may not form a circular graph`):
`@axiom/mcp-tools` already depends on `@axiom/cli-commands` (for
`runIndexRebuild`/`runIndexValidate`/`runValidateChanges`, pre-existing,
`cli-backed-handlers.ts`); adding `@axiom/mcp-server` (which depends on
`@axiom/mcp-tools` for `invokeMcpTool`) as a NEW dependency of
`@axiom/cli-commands` closes the cycle:
`cli-commands -> mcp-server -> mcp-tools -> cli-commands`.

Resolution: `runMcpServe` + `registerMcpServe` live in a NEW, APP-OWNED
file, `apps/cli/src/commands/mcp-serve.ts` — NOT included in
`packages/cli-commands/tsconfig.json`'s `include` list (unlike `mcp.ts`),
so it compiles as part of `apps/cli` itself, which is a leaf in the
project-reference graph (nothing references `apps/cli` except the root
`tsconfig.json`). `registerMcpServe(program)` is called from
`apps/cli/src/index.ts` right after `registerMcp(program)`, and looks up
the already-registered `mcp` commander sub-command by name
(`program.commands.find(c => c.name() === 'mcp')`) to attach `serve` to
the same group rather than creating a duplicate one. This preserves the
existing `mcp-tools -> cli-commands` edge untouched and adds no new edge
into that cycle.

### Files changed/added

- Added: `packages/mcp-server/{package.json,tsconfig.json}`,
  `packages/mcp-server/src/{protocol,context,tool-sets,input-builders,
  server,stdio,index}.ts`, `packages/mcp-server/tests/{server,stdio}.test.ts`.
- Added: `apps/cli/src/commands/mcp-serve.ts`,
  `apps/cli/tests/mcp-serve.test.ts`.
- Modified: `Axiom/tsconfig.json` (root, `+packages/mcp-server` reference),
  `Axiom/vitest.config.ts` (`+@axiom/mcp-server` alias),
  `apps/cli/tsconfig.json` (`+@axiom/mcp-server` path + reference),
  `apps/cli/package.json` (`+@axiom/mcp-server` dependency),
  `apps/cli/src/index.ts` (`+registerMcpServe` import + call),
  `apps/cli/src/commands/mcp.ts` (header-comment-only: documents why
  `serve` is NOT here; no behavior change to `list`/`validate`/`repair`/
  `inventory`).

### npm workspaces symlink gotcha (environment, not code)

A NEW workspace package (`packages/mcp-server/`) is not automatically
symlinked into the root `node_modules/@axiom/` until `npm install` (or
`npm install --workspaces`) re-runs — `tsc -b` and `vitest` both resolve
via the alias/`paths` maps and never noticed, but the actually-compiled
CLI binary (`node apps/cli/dist/index.js`) failed at runtime with
`Cannot find module '@axiom/mcp-server'` until this was done. The
pre-existing `@axiom/mcp-tools` (added by an earlier increment) had the
exact same gap, confirming this is a standing environment step, not
something introduced by this increment. Fixed here with an offline,
workspace-only relink (`npm install --workspaces --offline --no-audit
--no-fund`) — no network/registry access, consistent with the "no new npm
dependencies, no network install assumed" guardrail. Verified the real
spawned binary afterward: `node apps/cli/dist/index.js mcp serve --kind
sdd --project-root <repo>` piped an `initialize` + `tools/list` request
and returned the correct newline-delimited JSON-RPC responses.

## Validation

### `npm run build` (tsc -b), from `Axiom/`

```
> axiom-product@0.1.0 build
> tsc -b

(clean exit, no errors; re-run confirmed idempotent)
```

### `npx vitest run packages/mcp-server apps/cli packages/mcp-tools`

```
 Test Files  64 passed (64)
      Tests  527 passed (527)
```

New tests: `packages/mcp-server/tests/server.test.ts` (11), `.../stdio.test.ts`
(5), `apps/cli/tests/mcp-serve.test.ts` (2) — all passing, zero failures,
zero regressions in the target scope.

### Full-repo `npx vitest run` (extra check, not required by the increment)

```
 Test Files  10 failed | 165 passed (175)
      Tests  10 failed | 1799 passed (1809)
```

The 10 failures are pre-existing and unrelated to this increment's diff —
confirmed by re-running each failing file in isolation (same 10 fail,
same way) and by inspecting each: they all fail reading this workspace's
own dogfooded fixtures (`axiom.config/skills-catalog.yaml`,
`axiom.config/agents-catalog.yaml`, the real `axiom.spec/` model-routing
policy, `resolveProject`'s real-repo scope test, `runDoctorChecks`'
real-repo test) or are order/environment-sensitive
(`audit-trail-sink.test.ts`'s retention sweep,
`repair-add-gitignore.test.ts`). None touch `@axiom/mcp-server`,
`@axiom/mcp-tools`, `@axiom/cli-commands`, or any file this increment
changed. `packages/skills/tests/catalog.test.ts` is the one explicitly
named as a known pre-existing failure in the increment brief; the other 9
are the same class of pre-existing dogfood/environment failure, not new
regressions.

## Result

Implemented as scoped: `@axiom/mcp-server` (JSON-RPC 2.0 dispatcher +
newline-delimited stdio transport + best-effort context resolution +
per-capability input builders covering all 17 `@axiom/mcp-tools`
capabilities) and `axiom mcp serve --kind sdd|spec` are both real and
tested. The only deviation from the original plan is the CLI file
placement (`mcp-serve.ts` instead of inline in `mcp.ts` /
`@axiom/cli-commands`'s barrel), forced by a genuine project-reference
cycle discovered during the build step and resolved by keeping the new
code app-side, per "Implementation notes" above. All acceptance criteria
are met; build and targeted test validation are green.

## General spec integration

No `general-spec.md`-equivalent stable-knowledge doc exists yet in
`Axiom.Spec/specs/` beyond the numbered `0X_*.md` files (out of scope to
restructure here per the "do not touch 00-08" guardrail). The stable
knowledge this increment introduces — that `@axiom/mcp-server` is now the
runnable implementation behind the `sdd-mcp-server`/`spec-mcp-broker` ids
referenced in `06_Integraciones_y_Capacidades.md` — is recorded here and in
this package's own source comments; a future increment that revisits
`06_Integraciones_y_Capacidades.md` should link back to this package.

### Integración canónica realizada (pase de round 4, 2026-07-08)

El pase de integración de round 4 (orquestador) integró el conocimiento
estable de este incremento en los ficheros canónicos `00–08`:

- **`06_Integraciones_y_Capacidades.md`** (PRIMARIO): nueva subsección
  "Server MCP ejecutable (`@axiom/mcp-server` + `axiom mcp serve`)" que
  supersede la afirmación previa de "sin server MCP genérico todavía"
  (item 3 de "Integraciones esperadas" también actualizado) — documenta
  el protocolo JSON-RPC 2.0 sobre stdio, el split 7 `sdd.*` / 10 `spec.*`
  por `--kind`, `resolveMcpServerContext` best-effort, el seam de build
  (`packages/mcp-server/` + `apps/cli/src/commands/mcp-serve.ts`) y la
  ausencia de dependencias npm nuevas.
- **`05_Interfaces_Operativas.md`**: `axiom mcp serve --kind <sdd|spec>
  --project-root <path>` añadido a la superficie de comandos ampliada.
- **`08_Glosario.md`**: término "`@axiom/mcp-server` / `axiom mcp serve`"
  añadido en la nueva sección de tanda 4.
