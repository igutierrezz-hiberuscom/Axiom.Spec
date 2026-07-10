# Increment: MCP native config mapping

Status: closed
Date: 2026-07-08

## Goal

`runWorkspaceSetup` writes `.axiom/mcp.yml` (canonical) plus a CUSTOM-shape
`.opencode/mcp.json` / `.claude/mcp.json` (`{schemaVersion, projectId,
servers:[...]}`). That custom shape is NOT what a real opencode/Claude
Code/Cursor/VS Code install reads to auto-launch MCP servers. This
increment maps Axiom's two MCP entries (`sdd-mcp-server`,
`spec-mcp-broker`) onto each tool's REAL NATIVE MCP-config schema, written
into every repo of the workspace, so a real tool install actually launches
the servers.

## Context

Read first: `apps/cli/src/commands/workspace-mcp.ts`
(`buildWorkspaceMcpServers`, `writeWorkspaceMcpConfig`),
`apps/cli/src/commands/workspace-adapters.ts` (`generateWorkspaceAdapters`
— adapters are generated per-repo per selected adapter),
`packages/user-workspace/src/mcp-config.ts` (`McpServerEntry`), and
`apps/cli/tests/e2e/workspace-mcp.e2e.test.ts`. Prior increments
(`INC-20260708-mcp-runnable-server`, `INC-20260708-mcp-launch-config-wiring`,
both archived) shipped the runnable `axiom mcp serve --kind <sdd|spec>
--project-root <path>` server and wired `command`/`args` into the two
managed entries, but explicitly scoped OUT mapping to each tool's native
schema (see that increment's "Non-goals").

### Verified native schemas (from official docs)

For an Axiom MCP entry `{ id, command:'axiom', args:[...], env? }`
(`id` ∈ `{sdd-mcp-server, spec-mcp-broker}`):

- **claude-code** → `<repoRoot>/.mcp.json` →
  `{ "mcpServers": { "<id>": { "command": "axiom", "args": [...], "env": {...} } } }`
  (env optional, omitted when empty).
- **cursor** → `<repoRoot>/.cursor/mcp.json` → same shape as claude-code
  (`{ "mcpServers": { "<id>": { command, args, env? } } }`).
- **copilot-vscode** and **github-copilot** → `<repoRoot>/.vscode/mcp.json`
  → `{ "servers": { "<id>": { "type": "stdio", "command": "axiom", "args": [...], "env": {...} } } }`
  (key is `servers`, not `mcpServers`; `type:'stdio'` required).
- **opencode** → `<repoRoot>/opencode.json` →
  `{ "$schema": "https://opencode.ai/config.json", "mcp": { "<id>": { "type": "local", "command": ["axiom", ...args], "enabled": true, "environment": {...} } } }`
  (`command` is a single array `[command, ...args]`; `env` renamed
  `environment`; `type:'local'`, `enabled:true`).
- **antigravity**, **visual-studio-2026**, **litellm** → no verified
  native MCP schema exists → emit a warning naming the target (same
  pattern as the existing "adapter projection not available" message);
  never invent a schema.

## Scope

1. New module `apps/cli/src/commands/native-mcp-config.ts`: one emitter
   function per native target family (claude-code/cursor shape,
   copilot-vscode/github-copilot shape, opencode shape) plus a
   `writeNativeMcpConfig({ target, repoRoot, servers })` dispatcher.
   MERGE-PRESERVING: parses the existing native file if present, replaces
   only the Axiom-managed server entries (keyed by entry id) inside the
   tool's server map, and preserves every other key/server already
   present. Creates a minimal valid file when absent. Atomic write
   (tmp+rename, reusing `atomicWriteFile` from `init.ts`). Unsupported
   targets (antigravity/visual-studio-2026/litellm) produce a warning and
   no file — never invent a schema.
2. Wire `writeNativeMcpConfig` into the workspace setup flow so it runs
   for EVERY repo of the workspace (control + spec + each role repo), for
   each selected adapter — a dev opening any repo should see the
   servers. Best-effort per repo/adapter: a failure never fails the
   setup; collect warnings + `filesCreated`.
3. Stop writing the OLD custom-shape `.opencode/mcp.json` /
   `.claude/mcp.json` from the workspace setup path
   (`writeWorkspaceMcpConfig` no longer calls
   `generateOpencodeMcpJson`/`generateClaudeCodeMcpJson`) — they are
   superseded by the native files at different paths (`opencode.json`,
   `.mcp.json`). `generateOpencodeMcpJson`/`generateClaudeCodeMcpJson`
   themselves stay in their packages (other code/tests may still
   reference them); only the workspace-setup call site changes.
   `.axiom/mcp.yml` remains the canonical, unchanged source.
4. Update tests: new unit tests for the native emitters (exact shape +
   path per target, merge-preservation, opencode's array/rename/flags,
   vscode's `servers`+`type:'stdio'`, unsupported-target warning);
   `apps/cli/tests/workspace-mcp.test.ts` +
   `apps/cli/tests/workspace-setup.test.ts` updated to assert native
   files instead of the old custom-shape files; the e2e
   (`apps/cli/tests/e2e/workspace-mcp.e2e.test.ts`) updated to assert
   native files exist with the right shape and that the live-spawn test
   reads its launch args from the native config.

## Non-goals

- No changes to `@axiom/mcp-server` or `axiom mcp serve`.
- No changes to `configure`/`sync`/`runInit`.
- No new npm dependencies.
- No invented native schema for antigravity/visual-studio-2026/litellm.
- No changes to `00`–`08` canonical spec docs (this increment's
  general-spec integration is deferred to the same follow-up round the
  prior two MCP increments used).

## Acceptance criteria

- [x] `writeNativeMcpConfig` emits the exact verified shape per target
      (claude-code/cursor `.mcp.json`/`.cursor/mcp.json` with
      `mcpServers`; copilot-vscode/github-copilot `.vscode/mcp.json` with
      `servers`+`type:'stdio'`; opencode `opencode.json` with
      `mcp`+array `command`+`environment`+`type:'local'`+`enabled`).
- [x] Merge-preserving: a pre-existing native file with a user's own
      server entry and other top-level keys keeps that content untouched;
      only the Axiom-managed ids are added/replaced.
- [x] Unsupported targets (antigravity, visual-studio-2026, litellm)
      produce a warning naming the target and write no native file.
- [x] Native configs are written per selected adapter, per repo, in
      every repo of the workspace (control + spec + role repos).
- [x] The old custom-shape `.opencode/mcp.json` / `.claude/mcp.json` are
      no longer written from the workspace setup path.
      `.axiom/mcp.yml` stays canonical and unchanged in shape.
- [x] The e2e live-spawn test reads its launch args from the native
      config file and the spawned servers still answer over stdio
      JSON-RPC.
- [x] `npm run build` (tsc -b) clean.
- [x] `npx vitest run apps/cli packages/adapters packages/user-workspace`
      and `npx vitest run apps/cli/tests/e2e` pass (known pre-existing
      dogfood failures out of scope).

## Open questions

None blocking — resolved under "Assumptions".

## Assumptions

- Home for the new emitters: `apps/cli/src/commands/native-mcp-config.ts`
  (app layer), matching where `workspace-mcp.ts`/`workspace-adapters.ts`
  already live, rather than extending each adapter package — keeps the
  target-dispatch enum in one place instead of duplicating it across
  five adapter packages.
- `command` stays the literal string `'axiom'` (CLI on PATH), same
  documented assumption as `INC-20260708-mcp-launch-config-wiring`.
- Merge-preservation is keyed strictly by entry id inside the
  tool-specific server map; unrelated keys anywhere else in the parsed
  JSON document are round-tripped byte-for-byte via re-serialization
  (whitespace/ordering may normalize through `JSON.parse`/`stringify`,
  which is an accepted, unavoidable tradeoff of a merge-write and matches
  how the pre-existing custom-shape generators already behave).
- A native file that fails to parse (malformed pre-existing JSON) is
  treated as best-effort-failed for that repo/adapter: a warning is
  recorded and the file is left untouched (never clobbered), consistent
  with "never clobber a user's existing config."

## Implementation notes

- `apps/cli/src/commands/native-mcp-config.ts` (new): exports
  `NATIVE_MCP_TARGETS` (the subset of `AdapterTarget` with a verified
  native schema: `claude-code`, `cursor`, `copilot-vscode`,
  `github-copilot`, `opencode`), a `NativeMcpServerInput` type mirroring
  `McpServerEntry`'s launch-relevant fields (`id`, `command`, `args`,
  `env`), and `writeNativeMcpConfig({ target, repoRoot, servers })` which
  dispatches to one of three shape writers:
  - `writeMcpServersShapeFile` (claude-code → `.mcp.json`, cursor →
    `.cursor/mcp.json`): reads/parses existing file if present (falls
    back to `{}` on missing/malformed with a warning), ensures
    `mcpServers` object exists, sets/replaces each Axiom entry by id as
    `{ command, args, ...(env ? {env} : {}) }`, atomic write.
  - `writeVscodeMcpConfig` (copilot-vscode, github-copilot →
    `.vscode/mcp.json`): same merge pattern under the `servers` key, each
    entry `{ type: 'stdio', command, args, ...(env ? {env} : {}) }`.
  - `writeOpencodeNativeConfig` (opencode → `opencode.json`): merge under
    `mcp`, each entry `{ type: 'local', command: [command, ...args],
    enabled: true, ...(env ? {environment: env} : {}) }`; ensures
    `$schema: 'https://opencode.ai/config.json'` is present when creating
    a new file (preserved as-is if already present in an existing file).
  - Unsupported targets (`antigravity`, `visual-studio-2026`, `litellm`):
    dispatcher returns a warning, writes nothing.
  All three writers are best-effort (try/catch internally); errors never
  throw out of `writeNativeMcpConfig`.
- `apps/cli/src/commands/workspace-mcp.ts`: `writeWorkspaceMcpConfig` no
  longer imports/calls `generateOpencodeMcpJson` /
  `generateClaudeCodeMcpJson`; `MCP_CAPABLE_ADAPTERS` and the old
  adapter-projection loop removed from this function (it now only writes
  `.axiom/mcp.yml`). A new function `writeWorkspaceNativeMcpConfigs`
  (same file) loops over every repo × every selected adapter and calls
  `writeNativeMcpConfig`, collecting `filesCreated`/`warnings`.
- `apps/cli/src/commands/workspace-setup.ts`: step 4 (MCP generation)
  updated to call `writeWorkspaceNativeMcpConfigs` with
  `spec.repos.map((r) => ({ roleKey: r.roleKey, path: r.path }))` (same
  repo list shape `generateWorkspaceAdapters` already receives) after
  `.axiom/mcp.yml` is written, appending its `filesCreated`/`warnings`.
- Old custom-shape generators (`generateOpencodeMcpJson`,
  `generateClaudeCodeMcpJson`) left untouched in their packages — no
  longer referenced from the workspace-setup path, but still exported and
  still covered by their own package-level tests.

## Validation

### `npm run build` (tsc -b), from `Axiom/`

```
> axiom-product@0.1.0 build
> tsc -b

(clean exit, no errors)
```

### `npx vitest run apps/cli packages/adapters packages/user-workspace`

```
 Test Files  72 passed (72)
      Tests  656 passed (656)
```

All targeted suites green, including 15 new `native-mcp-config.test.ts`
unit tests, the rewritten `workspace-mcp.test.ts` (12 tests: standalone
`writeWorkspaceMcpConfig` mcp.yml-only, standalone
`writeWorkspaceNativeMcpConfigs`, end-to-end wiring via
`runWorkspaceSetup`), and updated assertions in `workspace-setup.test.ts`
(16 tests) and `tui.test.ts` (both updated to assert native files instead
of the old custom-shape files).

### `npx vitest run apps/cli/tests/e2e`

```
 Test Files  1 passed (1)
      Tests  4 passed (4)
```

Part A asserts the native files (`.mcp.json` and `opencode.json`) exist
with the correct schema in EVERY workspace repo (control, spec, backend,
custom "data" role) and that the old custom-shape
`.opencode/mcp.json`/`.claude/mcp.json` are absent. Part B's live spawn
now reads its launch args from the native `.mcp.json` file
(`mcpServers['sdd-mcp-server'/'spec-mcp-broker'].args`) and both the
sdd-mcp-server and spec-mcp-broker still answer real stdio JSON-RPC
(initialize, tools/list, tools/call, unknown-method -32601, clean exit).

## Result

Implemented as scoped. New module
`apps/cli/src/commands/native-mcp-config.ts` emits the verified native
MCP schema for claude-code/cursor (`mcpServers`), copilot-vscode/
github-copilot (`servers` + `type:'stdio'`), and opencode (`mcp` + array
`command` + `environment` + `type:'local'`/`enabled`), all
merge-preserving and atomic-write. `writeWorkspaceMcpConfig` now only
writes `.axiom/mcp.yml`; a new `writeWorkspaceNativeMcpConfigs`
(`workspace-mcp.ts`) writes the native config of every selected adapter
into every repo of the workspace. The old custom-shape
`.opencode/mcp.json`/`.claude/mcp.json` are no longer written from the
workspace setup path — `generateOpencodeMcpJson`/`generateClaudeCodeMcpJson`
remain in their packages, still covered by their own tests, just no
longer called from here. antigravity/visual-studio-2026/litellm degrade
to a warning, no invented schema. All acceptance criteria met; full
build + targeted test suites green.

## General spec integration

Integrated into the canonical spec (`Axiom.Spec/specs/00`–`08`) on
2026-07-08. This supersedes the earlier "deferred / no general-spec file
yet" note: the 00–08 canonical spec files ARE the stable-knowledge home,
and this increment's knowledge has now been folded into them (Spanish
prose, citing `INC-20260708-mcp-native-config-mapping`, append/supersede
rather than wholesale rewrite):

- **06_Integraciones_y_Capacidades.md** (primary): SUPERSEDED the prior
  "Caveat honesto (pendiente)" bullet in the "Server MCP ejecutable"
  section — mapping to each tool's native MCP-config schema is now DONE,
  not a pending follow-up. Added a new subsection **"Config MCP nativa
  por herramienta destino"** with the per-tool file+schema table
  (claude-code `.mcp.json`, cursor `.cursor/mcp.json`, copilot-vscode/
  github-copilot `.vscode/mcp.json` with `servers`+`type:'stdio'`,
  opencode `opencode.json` with array `command`+`environment`+
  `type:'local'`+`enabled`; antigravity/visual-studio-2026/litellm →
  warning, no file), the merge-preserving behavior (malformed JSON →
  warning, no clobber), per-repo×per-adapter emission across the whole
  workspace, `.axiom/mcp.yml` staying canonical, and the e2e proving the
  native config is runnable. Also updated the
  "INC-20260705-workspace-adapters-multiselect" projection bullet and the
  multi-adapter dispatch table (opencode/claude-code rows) to note the
  custom-shape `.opencode/mcp.json`/`.claude/mcp.json` are superseded by
  the native files.
- **05_Interfaces_Operativas.md**: updated the "Adapters — MCP config por
  proyecto" section — the workspace call site of
  `generateOpencodeMcpJson`/`generateClaudeCodeMcpJson` was retired; the
  workspace path now emits the native files (`opencode.json`, `.mcp.json`,
  `.cursor/mcp.json`, `.vscode/mcp.json`) instead of the custom-shape
  `.opencode/mcp.json`/`.claude/mcp.json`.
- **03_Modelo_Operativo_y_Datos.md**: updated the `mcp.yml` section — the
  custom-shape generators still exist/are tested but their workspace call
  site was retired; the "primer y único call site generador vivo"
  paragraph now lists the native config files (with schemas) that the
  workspace path emits, and notes `.axiom/mcp.yml` remains canonical and
  unchanged in shape.
- **08_Glosario.md**: added a **"Config MCP nativa por herramienta"** term
  (file+schema per target family, merge-preserving/atomic, unsupported
  targets → warning, supersedes the custom-shape files) and trimmed the
  now-resolved "mapeo al schema MCP NATIVO ... sigue siendo un follow-up"
  clause from the "Launch config de MCP" term.

Files left untouched (no contradiction): 00, 01, 02, 04, 07.
