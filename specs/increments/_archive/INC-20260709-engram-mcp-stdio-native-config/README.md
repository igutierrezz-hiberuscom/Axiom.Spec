# Increment: Engram MCP stdio native config

Status: closed
Date: 2026-07-09

## Goal

When the `engram` memory provider is selected for an Axiom workspace, emit a
LOCAL, project-pinned `engram` MCP server entry into each selected tool's
NATIVE MCP config — so OpenCode, Claude Code, Cursor, and the VS
Code-family adapters (Copilot VS Code / GitHub Copilot) reach `engram` over
MCP stdio, and no `engram serve` HTTP daemon is needed for that purpose.

## Context

`engram` ships two distinct launch modes, confirmed directly against
`C:/repos/engram/cmd/engram/main.go` (`cmdMCP`, roughly L847-905):

- `engram serve` starts the HTTP+MCP daemon on port 7437.
- `engram mcp` starts the MCP server over stdio only — no HTTP, no port.
  `cmdMCP` parses both the `--flag value` and `--flag=value` forms for
  `--project` and `--tools`, so the space-separated array form
  (`['mcp', '--project', projectId, '--tools', 'agent']`) is valid.

Axiom's workspace setup (`runWorkspaceSetup`, `workspace-setup.ts`) already
writes two kinds of MCP configuration once the engram provider selection
exists in scope:

- `.axiom/mcp.yml`, the canonical, project-scoped `McpProjectConfig`
  (`type:'axiom'` brokers only: `sdd-mcp-server` / `spec-mcp-broker`).
- Per-tool NATIVE MCP configs (`writeWorkspaceNativeMcpConfigs` /
  `writeNativeMcpConfig`, `native-mcp-config.ts`), one file per adapter per
  repo, in each tool's own real, verified schema.

Before this increment, `engram` was never wired into either of those, so an
OpenCode (or Claude Code / Cursor / VS Code) session in an Axiom workspace
had no way to reach `engram` except by manually running `engram serve` and
pointing a client at its HTTP endpoint — an extra daemon adapters should not
need.

## Scope

- `Axiom/apps/cli/src/commands/workspace-mcp.ts`:
  - Export `ENGRAM_NATIVE_MCP_SERVER_ID = 'engram'`.
  - Export a pure helper, `buildEngramNativeServer(projectId)`, returning
    `{ id: 'engram', command: 'engram', args: ['mcp', '--project',
    projectId, '--tools', 'agent'] }`.
  - Extend `WriteWorkspaceNativeMcpConfigsArgs` with an optional
    `engram?: { projectId: string }`.
  - `writeWorkspaceNativeMcpConfigs`: when `args.engram` is present, append
    the engram native server entry to the list of servers projected into
    every repo × adapter, reusing the existing `writeNativeMcpConfig`
    writers (one per tool shape) unmodified. The pre-existing
    zero-servers-guard now runs AFTER the engram entry is appended, so an
    engram-only workspace (zero `type:'axiom'` brokers) still gets the
    native entry written.
- `Axiom/apps/cli/src/commands/workspace-setup.ts`:
  - At the MCP-generation step, keep `.axiom/mcp.yml` writing gated on
    `servers.length > 0` exactly as before (engram is never written there).
  - Hoist the `writeWorkspaceNativeMcpConfigs` call so it runs when
    `servers.length > 0 || engramEnabled` (previously nested only inside
    the `servers.length > 0` branch), passing `engram: engramEnabled ? {
    projectId: effectiveProjectId } : undefined` alongside the pre-existing
    `repos` / `servers` / `adapters` arguments.
- `Axiom/apps/cli/tests/workspace-mcp.test.ts`: new tests covering
  `buildEngramNativeServer` directly, and `writeWorkspaceNativeMcpConfigs`
  with/without an `engram` argument, across all four native shapes
  (claude-code, cursor, copilot-vscode/github-copilot, opencode), asserting
  merge-preservation of pre-existing user servers and the absence of any
  HTTP/7437/`ENGRAM_URL` reference in the generated engram entry.

## Non-goals

- Do not add `engram` to `.axiom/mcp.yml` — that file is Axiom's own
  canonical `type:'axiom'` broker config; `engram` is a third-party local
  stdio MCP server, not an Axiom broker.
- Do not change how `engram` is otherwise launched elsewhere (e.g.
  `@axiom/memory`'s `engram-backend.ts` local process launch is untouched
  and out of scope).
- Do not touch the `engram` or `GentleAI` repositories.
- Do not add native MCP support for adapters that have no verified native
  MCP schema today (`antigravity` / `visual-studio-2026` / `litellm`) —
  unchanged, still degrade to a warning.
- No speculative/enterprise architecture beyond the scoped change.

## Acceptance criteria

- [x] When the `engram` provider is selected for a workspace, each selected
      tool's native MCP config in each workspace repo contains a LOCAL
      `engram` stdio entry pinned to the project id, in that tool's correct
      native shape.
- [x] The projection is merge-preserving: pre-existing user servers/keys in
      any native config file are never clobbered.
- [x] No generated Axiom config references `engram` via HTTP, port 7437, or
      an `ENGRAM_URL` environment variable.
- [x] `.axiom/mcp.yml` never contains an `engram` entry.
- [x] The `engram` entry is absent from every native config when the
      provider is not selected.
- [x] Build passes (`npm run build`, `tsc -b`, no errors).
- [x] Targeted tests pass (`workspace-mcp.test.ts`, `native-mcp-config.test.ts`,
      `workspace-setup.test.ts`).

## Open questions

None — decisions baked by orchestrator.

## Assumptions

- `engram mcp --project <id> --tools agent` (space-separated args) is a
  valid, stable invocation of the currently-installed `engram` binary,
  confirmed directly against its Go source (`cmdMCP`,
  `C:/repos/engram/cmd/engram/main.go` ~L847-905) rather than inferred from
  behavior alone.
- The whole Axiom workspace is treated as a single Axiom project for engram
  purposes: every repo in the workspace receives the SAME `engram` native
  entry, pinned to the same `effectiveProjectId` (project isolation is at
  the workspace/project level, not per-repo).
- `engram` is assumed resolvable on `PATH` wherever the generated native
  config is consumed (same assumption the existing `axiom` launch command
  already makes for the `sdd-mcp-server` / `spec-mcp-broker` entries).

## Implementation notes

`workspace-mcp.ts`:

- `ENGRAM_NATIVE_MCP_SERVER_ID = 'engram'` and `buildEngramNativeServer`
  were added near the top of the file, next to the other MCP id/helper
  constants, with a comment block explaining the stdio-vs-HTTP distinction
  and why this entry never appears in `.axiom/mcp.yml`.
- `writeWorkspaceNativeMcpConfigs` now builds `nativeServers` as a mutable
  array (typed `NativeMcpServerInput[]`, imported from `native-mcp-config`)
  so the engram entry can be `push`ed onto it conditionally, before the
  existing `nativeServers.length === 0` early-return guard runs. Everything
  downstream (the repo × adapter loop, best-effort try/catch,
  unsupported-adapter warning) was left untouched — it operates on
  `nativeServers` however it was built.

`workspace-setup.ts`:

- The MCP-generation `try` block's structure changed minimally: the
  `writeWorkspaceMcpConfig` call (which writes `.axiom/mcp.yml`) stayed
  inside its own `if (servers.length > 0)` block, unchanged. A new,
  sibling `if (servers.length > 0 || engramEnabled)` block now wraps the
  `writeWorkspaceNativeMcpConfigs` call (previously nested inside the same
  `if` as the mcp.yml write), passing the new `engram` argument. No other
  logic in `runWorkspaceSetup` was touched.

`workspace-mcp.test.ts`:

- Added a `buildEngramNativeServer` unit test asserting the exact expected
  object for `projectId: 'proj-x'`.
- Added a `writeWorkspaceNativeMcpConfigs` test that pre-seeds each of the
  four native files with an unrelated `user-server` entry, calls the
  function with `engram: { projectId: 'proj-x' }` across all four adapter
  shapes, and asserts: (a) each tool's file gains the `engram` entry in its
  correct shape; (b) the pre-existing `user-server` entry survives
  untouched; (c) the serialized `engram` entry itself (isolated from the
  rest of each document, since `opencode.json` legitimately carries an
  `https://` `$schema` URL elsewhere in the file) contains none of `7437`,
  `ENGRAM_URL`, or `http` case-insensitively.
- Added a test confirming no `engram` entry is written when `args.engram`
  is omitted.
- Added a test confirming the engram-only path (`servers: []`, `engram`
  present) still writes the native file — covering the hoisted guard in
  `writeWorkspaceNativeMcpConfigs`.

No changes were needed in `native-mcp-config.ts` — the existing per-tool
writers were reused exactly as-is, confirming they generalize cleanly to
any `NativeMcpServerInput`, not just the two Axiom broker ids they were
originally written for.

## Validation

From `C:/repos/Axiom Workspace/Axiom`:

- `npm run build` (`tsc -b`) — clean, no output, no errors.
- `npx vitest run apps/cli/tests/workspace-mcp.test.ts apps/cli/tests/native-mcp-config.test.ts apps/cli/tests/workspace-setup.test.ts`:

```
 ✓ apps/cli/tests/native-mcp-config.test.ts (15 tests) 71ms
 ✓ apps/cli/tests/workspace-mcp.test.ts (16 tests) 979ms
 ✓ apps/cli/tests/workspace-setup.test.ts (33 tests) 9832ms

 Test Files  3 passed (3)
      Tests  64 passed (64)
```

The previously-noted baseline (`packages/skills/tests/catalog.test.ts`) was
also re-run in isolation as a sanity check and passed (11/11) — no
pre-existing failure was observed in this session, and none of the changes
in this increment touch that package.

## Result

Implemented exactly the baked-in design: a pure `buildEngramNativeServer`
helper producing the stdio launch entry (`engram mcp --project <id>
--tools agent`), wired into `writeWorkspaceNativeMcpConfigs` via a new
optional `engram` argument that appends the entry before the
zero-servers guard, and threaded through `runWorkspaceSetup` so the native
projection now runs whenever either an Axiom broker or the engram provider
is present. `.axiom/mcp.yml` generation was left completely untouched
(still gated on `servers.length > 0`, still only the two `type:'axiom'`
broker entries). All existing native-config writers (`native-mcp-config.ts`)
were reused unmodified — no per-tool shape logic was hand-rolled for
engram. Build is clean; all targeted tests (new and pre-existing) pass.
All acceptance criteria are met.

## General spec integration

Integrada por el orchestrator en la pasada final cross-increment (batch
INC-20260709-*):

- `06_Integraciones_y_Capacidades.md` — nueva subsección "engram vía MCP
  stdio en la config nativa por herramienta (sin daemon `engram serve`) —
  INC-20260709-engram-mcp-stdio-native-config", documentando explícitamente
  que con engram sobre MCP stdio NO hace falta el daemon `engram serve`
  (puerto 7437) y que ninguna config generada referencia engram por
  HTTP/`ENGRAM_URL`.
- `05_Interfaces_Operativas.md` — nueva sección "Entrada MCP nativa de engram
  (stdio) en cada adapter".
- `03_Modelo_Operativo_y_Datos.md` — nota sobre la entrada MCP nativa de
  engram (forma por tool, project-pinned, fuera de `.axiom/mcp.yml`).
- `08_Glosario.md` — término "Entrada MCP nativa de engram (stdio, sin
  `engram serve`)".

Archivado en `Axiom.Spec/specs/increments/_archive/`.
