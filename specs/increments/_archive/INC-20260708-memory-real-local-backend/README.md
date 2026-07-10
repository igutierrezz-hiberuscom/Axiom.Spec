# Increment: Memory real local backend (AB5)

Status: closed
Date: 2026-07-08

## Goal

Make `@axiom/memory` REAL, LOCAL, cross-session, with topic-keyed upsert,
session summaries, and a real MCP surface — by wiring ENGRAM
(`C:\repos\engram`, a Go binary + local SQLite + FTS5, LOCAL MCP stdio) as
an additive `MemoryBackend` implementation, with graceful fallback to the
existing in-memory/JSON backend when engram is not installed locally.

## Context

- `@axiom/memory` (GATE 0024, Lote A/P0) was an MVP: in-memory `Map` +
  JSON file persistence (`.axiom-state/local/memory/<projectId>.json`),
  no full-text search, no topic-keyed upsert, no session lifecycle, no
  MCP surface. Its `MemoryBackend` interface was already deliberately
  duck-typed ("Lote B (P1) pueda sustituirla por un wrapper sobre Engram
  sin tocar el resto del runtime") to allow exactly this kind of backend
  swap.
- `memory.decisionRecall` / `memory.contextRecall` were declared
  capability ids (`axiom.config/capabilities.yaml`, both `required`) with
  a provider mapping in `axiom.config/providers.yaml` (`engram`, `kind:
  project-scoped-memory`, `capabilities: [memory.decisionRecall]`,
  `fallback: generated-snapshots`, `mvpDefault: false`), but NO real MCP
  tool handler existed in `@axiom/mcp-tools` — `packages/installer/src/
  registry.ts`'s `EXTERNAL_DEPS_BY_CAPABILITY` declared them with empty
  handler arrays.
- GATE 0024 cross-project isolation was already enforced in the in-memory
  backend (`createInMemoryBackend(projectRoot, scope)` pins to
  `scope.projectId`, rejects mismatches with `cross-project-blocked`) —
  this increment preserves and extends the SAME guarantee to the new
  engram backend and to the MCP surface.
- Prior increment AB3 (`INC-20260708-provider-runtime-execution-seam`)
  shipped `@axiom/providers`'s `createStdioMcpClient({command, args, cwd,
  env, timeoutMs})` (LOCAL child process, newline-delimited JSON-RPC 2.0
  over stdio, per-call timeout + clean shutdown) and the `LOCAL_ONLY`/
  `isLocalTarget` guard. AB4 (`INC-20260708-code-intel-providers-wired`)
  proved the pattern for codegraph/serena/graphify: one shared factory,
  one stub MCP server fixture per tool, `not-installed` vs `error`
  degrade classification.
- `C:\repos\engram`'s own README/docs (`docs/AGENT-SETUP.md`) and source
  (`internal/mcp/mcp.go`) were read directly to confirm the EXACT LOCAL
  MCP launch command, tool names, and parameter/response shapes (see
  "Engram MCP wire shape" below) — no assumption, no invented API.
- `INC-20260708-mcp-project-isolation-hardening` established the
  "pin-at-construction, reject-on-mismatch" pattern for `mcp-server`'s
  input builders (`pinned()`/`rejectCrossProject` in `input-builders.ts`)
  — this increment's `memory.*` input builder follows the SAME family of
  guarantee (see "Project isolation" below), adapted to memory's stricter
  "no legitimate cross-project use case at all" semantics.

## Scope

- `packages/memory/src/types.ts` — additive `MemoryEntry.topicKey`/
  `.sessionId`, `MemoryBackend.saveSessionSummary?` (optional method),
  new `MemorySessionSummary` type.
- `packages/memory/src/store.ts` — topic-key UPSERT semantics for
  `createInMemoryBackend`'s `save` (find-and-replace by
  `(projectId, topicKey)`), `saveSessionSummary` implementation (stored as
  an ordinary `MemoryEntry`, `kind: 'context'`, tag `session-summary`),
  new `saveMemorySessionSummary` helper, `saveMemory` now threads
  `topicKey`/`sessionId` through.
- `packages/memory/src/engram-backend.ts` (new) — `createEngramBackend`,
  a `MemoryBackend` that talks to a LOCAL `engram mcp` process via AB3's
  `createStdioMcpClient`, mapping Axiom memory ops to engram's real MCP
  tools (see "Engram MCP wire shape" below). Enforces GATE 0024 the same
  way as `createInMemoryBackend` (client-side validation) PLUS process-
  level pinning via `engram mcp --project=<projectId>`.
- `packages/memory/src/resolve-backend.ts` (new) — `resolveMemoryBackend`,
  probes whether `engram mcp` is spawnable (a real `initialize()`
  handshake, not just a PATH check) and picks engram-if-available else
  the JSON fallback backend. Never throws.
- `packages/memory/src/index.ts` — barrel exports for the above.
- `packages/memory/package.json` / `tsconfig.json` — new dependency on
  `@axiom/providers` (for `createStdioMcpClient`/`isLocalTarget`).
- `packages/memory/tests/fixtures/stub-engram-mcp-server.mjs` (new) —
  hermetic stub emulating `mem_save`/`mem_search`/`mem_context`/
  `mem_get_observation`/`mem_session_summary`, including topic_key upsert
  and `--project=` pin enforcement.
- `packages/memory/tests/engram-backend.test.ts` (new) — 11 tests: save/
  query round-trip, `load()`, topic-key upsert, session summary, GATE
  0024 (client-side + server-side), `not-installed` fallback,
  `resolveMemoryBackend`'s engram/json/forceJson paths.
- `packages/mcp-tools/src/memory-handlers.ts` (new) — `decisionRecall`/
  `contextRecall` real handlers backed by `resolveMemoryBackend` +
  `queryMemory` + `buildRecallResult`.
- `packages/mcp-tools/src/registry.ts` / `types.ts` / `index.ts` — register
  the two new capability ids; `McpToolHandler` now additionally allows a
  `Promise<McpToolResult<T>>` return (additive, existing sync handlers
  unaffected).
- `packages/mcp-tools/package.json` / `tsconfig.json` — new dependency on
  `@axiom/memory`.
- `packages/mcp-tools/examples/{capabilities,providers}.example.yaml` +
  `tests/{registry,capability-routing-roundtrip}.test.ts` — updated for
  19 capability ids (was 17) and the new `memory` domain routing through
  `engram` (not `axiom-gateway`).
- `packages/mcp-server/src/tool-sets.ts` — new `'memory'` `McpServerKind`,
  `MEMORY_TOOL_CAPABILITY_IDS`, tool descriptions.
- `packages/mcp-server/src/input-builders.ts` — `buildMemoryRecall`,
  ALWAYS pins `projectRoot` to `context.projectRoot` (caller-supplied
  `projectRoot`/`projectId` args are never read, not merely rejected on
  mismatch).
- `packages/mcp-server/src/server.ts` / `stdio.ts` — `McpServer.handle()`
  is now `async` (`Promise<JsonRpcResponse | undefined>`) to support the
  memory domain's genuinely-asynchronous handlers; `stdio.ts` chains
  dispatch onto a promise queue to preserve response ordering.
- `packages/mcp-server/tests/server.test.ts` — 19 existing call sites
  updated to `await`; 5 new tests for the `memory` kind (serverInfo name,
  tool list, decisionRecall/contextRecall dispatch + project pinning,
  cross-kind rejection).
- `apps/cli/src/commands/mcp-serve.ts` — `--kind` now accepts `"memory"`.
- `apps/cli/tests/mcp-serve.test.ts` — 2 new scenarios (memory kind
  init/tools-list round-trip; `tools/call memory.decisionRecall` via
  injected streams, foreign `projectRoot` arg ignored).
- `packages/providers/tests/stdio-mcp-client.test.ts` — tiny drive-by:
  bumped the hard-coded 200ms timeout to 2000ms (flaky under parallel CPU
  load).
- **Workspace-level fix (discovered during validation, not originally in
  scope but required for correctness)**: ran `npm install` to relink
  `node_modules/@axiom/providers` — the package existed on disk (added by
  AB3) but had never been linked into `node_modules/@axiom/`, so the
  compiled CLI `dist/` failed at real Node `require()` time
  (`Cannot find module '@axiom/providers'`) as soon as `@axiom/memory`
  (now transitively required by `@axiom/cli-commands`'s `mcp.ts`) reached
  it. Vitest's own module resolution (path-mapped via tsconfig) masked
  this in every unit test; only the REAL spawned-binary E2E test
  (`apps/cli/tests/e2e/workspace-mcp.e2e.test.ts`) exercises actual
  `node dist/index.js`, which is what surfaced it.

## Non-goals

- No native SQLite npm dependency — engram's own local SQLite+FTS5 is
  used exclusively via its MCP stdio surface (per the explicit decision
  in the brief).
- No pooled/long-lived `engram mcp` process — mirrors AB4's code-intel
  clients: `createEngramBackend` spawns a fresh process per method call
  (`save`/`load`/`query`/`saveSessionSummary`), tears it down in a
  `finally`. A pooled/session-scoped variant is a possible future
  optimization, not required by the `MemoryBackend` contract.
- No changes to `@axiom/tool-routing`/`@axiom/capability-model` source
  code — `providers.yaml`/`capabilities.yaml` already declared the
  `engram` provider and the two `memory.*` capability ids before this
  increment; only `@axiom/mcp-tools`'s registration layer needed a real
  handler.
- No `axiom memory` CLI changes to switch backends yet — `apps/cli/src/
  commands/memory.ts`'s existing `show`/`add`/`query`/`inventory`
  subcommands still use `createInMemoryBackend` directly (unchanged).
  Wiring the CLI to `resolveMemoryBackend` is a natural, small follow-up
  but was not explicitly requested and is not required by the acceptance
  criteria below (the MCP surface is the requested integration point for
  this increment).
- No integration into `Axiom.Spec/specs/00-08` — per the batch brief,
  reserved for the orchestrator's final consolidation pass.
- No git commits.

## Acceptance criteria

- [x] `createEngramBackend(projectRoot, scope, opts?)` exists in
      `@axiom/memory`, implements the SAME `MemoryBackend` interface as
      `createInMemoryBackend`, and talks to engram's LOCAL MCP stdio
      server via `createStdioMcpClient`.
- [x] `createEngramBackend` maps Axiom memory ops to engram's real MCP
      tools: save → `mem_save` (with `project`/`topic_key`/`session_id`),
      query/load → `mem_search`, session summary → `mem_session_summary`.
      (`mem_context`/`mem_get_observation` are confirmed-compatible but
      not required by any `MemoryBackend` method signature — `mem_search`
      alone satisfies `load`/`query`'s contract with `query: ''`.)
- [x] GATE 0024 is enforced the SAME way as `createInMemoryBackend`:
      `scope.projectId` is pinned at construction (both client-side via
      argument validation AND process-level via `--project=`), and any
      mismatched `projectId` is rejected with `cross-project-blocked`
      before a single MCP call is made.
- [x] Graceful fallback: `resolveMemoryBackend(projectRoot, scope, opts?)`
      probes engram (a real `initialize()` handshake) and falls back to
      `createInMemoryBackend` when engram is not spawnable — never
      throws, always returns a one-line `note`.
- [x] `topicKey`/session fields added additively to `MemoryEntry`/
      `MemoryBackend`; topic-keyed UPSERT implemented for BOTH backends
      (engram native `topic_key`; JSON backend: find-and-replace by
      `(projectId, topicKey)`).
- [x] `memory.decisionRecall`/`memory.contextRecall` wired to REAL
      handlers in `@axiom/mcp-tools` (previously empty handler arrays),
      backed by `resolveMemoryBackend` + `queryMemory` +
      `buildRecallResult`.
- [x] The two capability ids are exposed in `mcp-server`'s tool sets (new
      `'memory'` `McpServerKind`) — decision justified in
      `tool-sets.ts`'s header comment (memory handlers are the first
      genuinely-async handlers in the package; folding into `sdd` would
      force every `sdd.*` caller through an async path for no benefit).
- [x] Memory MCP tools are pinned to the server's own `context.
      projectRoot` — no caller-supplied cross-project path/id is ever
      read (`buildMemoryRecall` in `input-builders.ts`), mirroring
      `INC-20260708-mcp-project-isolation-hardening`'s guard family.
- [x] Hermetic tests: stub engram MCP server fixture; save/recall
      round-trip; topic-key upsert; session summary; cross-project
      blocked (client-side AND server-side); fallback to JSON when
      engram absent; MCP handlers return results and are project-pinned
      (foreign projectId/projectRoot ignored, not merely rejected).
      NONE depend on the real engram binary being installed.
- [x] `npm run build` (`tsc -b`) clean.
- [x] `npx vitest run packages/memory packages/mcp-tools packages/mcp-server
      packages/providers apps/cli` passes (77 files / 666 tests).
- [x] `npx vitest run` (full, parallel) and `npx vitest run
      --no-file-parallelism` (full, serial) both pass with 0 regressions:
      `186/186` files, `1936/1936` tests, in BOTH modes.
- [x] AB3's flaky `stdio-mcp-client.test.ts` timeout bumped 200ms → 2000ms.

## Open questions

None blocking. Ambiguities resolved under Assumptions.

## Assumptions

- **`mem_session_summary` has no `project` argument** (confirmed in
  `internal/mcp/mcp.go`: "project field intentionally not read —
  auto-detect only, REQ-308 write-tool contract"). `createEngramBackend`
  therefore relies ENTIRELY on the process-level `--project=` pin (set at
  `launch()` time) to scope session-summary writes — there is no
  redundant argument-level check possible for this one tool, unlike
  `save`/`query` which do pass an explicit `project` argument as
  belt-and-suspenders.
- **`mem_search` as the single read primitive for both `load()` and
  `query()`**: engram exposes no "list everything" tool; `mem_search`
  with an empty `query` string and a generous `limit` is engram's own
  documented pattern for "browse recent entries for a project" (the Go
  handler never rejects an empty query — only the JSON Schema marks it
  "required" as a hint). `mem_context` was considered for `load()` but
  rejected: its envelope carries only human-readable text, no structured
  `results` array, making it unsuitable for reconstructing `MemoryEntry`
  objects.
- **Kind mapping (`MemoryKind` <-> engram's free-form `type` string)**:
  engram's `type` field is NOT a closed enum (its own docs list
  `decision, architecture, bugfix, pattern, config, discovery, learning`
  as *suggestions*). Chose the most direct round-trip:
  `decision`->`decision`, `bug`->`bugfix`, `learning`->`learning`,
  `pattern`->`pattern`, `context`->`manual` (engram's own save default).
  Reverse mapping (`engramTypeToKind`) is tolerant of any string
  (including values a human wrote directly via `mem_save`, e.g.
  `"architecture"`), falling back to a caller-configurable
  `fallbackKind` (default `'context'`).
- **`memory` as its own `McpServerKind`, not folded into `sdd`**: the
  `memory.*` handlers are the first genuinely-asynchronous handlers in
  `@axiom/mcp-tools` (they resolve a `MemoryBackend`, possibly spawning a
  LOCAL subprocess). A dedicated kind keeps the sync/async boundary
  aligned with server kind (every `sdd.*`/`spec.*` handler stays
  synchronous) and mirrors `providers.yaml`'s own modeling (`engram` is a
  distinct provider `kind: project-scoped-memory`, never folded into
  `axiom-gateway`). `McpServer.handle()` becoming `async` project-wide
  (rather than only for the `memory` kind) was the simplest correct
  option — ONE dispatch path instead of a sync/async fork inside
  `handleToolsCall`; a resolved value awaits instantly, so `sdd.*`/
  `spec.*` behavior is unchanged (confirmed by the full, unmodified
  passing test suite plus 19 mechanically-`await`ed pre-existing
  assertions in `server.test.ts`).
- **`memory.*` input-builder pinning is STRICTER than `pinned()`**: every
  other domain's builders (`sdd.*`/`spec.*`) allow a caller to
  explicitly re-assert the SAME value as the bound context (only a
  DIFFERENT value is rejected). Memory has no legitimate cross-project
  recall use case at all, so `buildMemoryRecall` simply never reads a
  caller-supplied `projectRoot`/`projectId` — there is nothing to
  validate-and-reject, only a value to ignore. This is documented inline
  in `input-builders.ts` as an intentional divergence from the
  established `pinned()` pattern, not an oversight.
- **No `axiom memory` CLI wiring to `resolveMemoryBackend` in this
  increment**: the brief's explicit deliverables were the backend +
  fallback factory + topic-key/session API + MCP surface; the existing
  CLI subcommands (`show`/`add`/`query`/`inventory`) continue to use
  `createInMemoryBackend` directly. This is a clean, small follow-up
  (swap one function call + make the CLI actions `async`, which they
  already are) left for a future increment rather than silently expanded
  scope.
- **Session summaries stored as ordinary `MemoryEntry` records
  (`kind: 'context'`, tag `session-summary`) in the JSON backend**: kept
  `saveSessionSummary` as its OWN `MemoryBackend` method (for API clarity
  and engram parity — engram has a distinct `mem_session_summary` tool),
  but the JSON backend's storage shape did not need a second file/format;
  reusing the existing `MemoryEntry` array keeps `loadMemory`/
  `queryMemory` able to surface session summaries alongside every other
  entry without new plumbing.

## Implementation notes

**Engram MCP wire shape** (confirmed by reading `C:\repos\engram`'s
`docs/AGENT-SETUP.md` and `internal/mcp/mcp.go` directly):

- **Launch command**: `engram mcp` (stdio transport). Documented for
  every agent integration (Claude Code, OpenCode, Cursor, VS Code,
  Windsurf, Gemini CLI, Codex, etc.) identically:
  `{"command": "engram", "args": ["mcp"]}`. Optional flags used by this
  increment: `--project=<name>` (pins the ENTIRE process to one project
  for its lifetime, takes precedence over cwd/session detection —
  confirmed in `docs/AGENT-SETUP.md`'s "Project detection in VS Code,
  WSL, and CI" section) and `--tools=agent` (the 16-tool agent-facing
  profile, vs. 20 tools bare).
- **Tool -> Axiom-op mapping**:
  | Axiom op | Engram tool | Key params used |
  |---|---|---|
  | `save` | `mem_save` | `title`, `content`, `type`, `project`, `topic_key?`, `session_id?`, `capture_prompt: false` |
  | `load`/`query` | `mem_search` | `query` (empty string for `load`), `project`, `limit` |
  | `saveSessionSummary` | `mem_session_summary` | `content`, `session_id?` (NO `project` arg — auto-detect only, scoped via the process `--project=` pin) |
  | (available, not required by `MemoryBackend`) | `mem_get_observation` | `id` — full untruncated content by id |
  | (available, not required) | `mem_context` | recent cross-session text context |
- **Response envelope**: every write/read tool wraps its
  `content[0].text` in a JSON envelope via engram's own
  `respondWithProject` helper: `{project, project_source, project_path,
  result: <human text>, ...extra}`. `mem_save`'s `extra` includes `id`/
  `sync_id`/`state`; `mem_search`'s `extra.results` is a STRUCTURED array
  (`{id, sync_id, title, type, state, scope, pinned, project?}`) — this
  is what `createEngramBackend` parses (`JSON.parse(content[0].text)`)
  rather than regex-scraping the human-readable text.

**Files created:**
- `Axiom/packages/memory/src/engram-backend.ts`
- `Axiom/packages/memory/src/resolve-backend.ts`
- `Axiom/packages/memory/tests/fixtures/stub-engram-mcp-server.mjs`
- `Axiom/packages/memory/tests/engram-backend.test.ts`
- `Axiom/packages/mcp-tools/src/memory-handlers.ts`

**Files edited:**
- `Axiom/packages/memory/src/{types,store,index}.ts`,
  `package.json`, `tsconfig.json`
- `Axiom/packages/mcp-tools/src/{types,registry,index}.ts`,
  `package.json`, `tsconfig.json`,
  `examples/{capabilities,providers}.example.yaml`,
  `tests/{registry,capability-routing-roundtrip}.test.ts`
- `Axiom/packages/mcp-server/src/{tool-sets,input-builders,server,stdio}.ts`,
  `tests/server.test.ts`
- `Axiom/apps/cli/src/commands/mcp-serve.ts`,
  `Axiom/apps/cli/tests/mcp-serve.test.ts`
- `Axiom/packages/providers/tests/stdio-mcp-client.test.ts` (timeout bump)
- Workspace: `npm install` (relinked `node_modules/@axiom/providers`,
  pre-existing gap unrelated to any specific file edit)

## Validation

- `npm run build` (`tsc -b`, from `Axiom/`): clean, no errors.
- `npx vitest run packages/memory packages/mcp-tools packages/mcp-server
  packages/providers apps/cli`:
  ```
  Test Files  77 passed (77)
       Tests  666 passed (666)
  ```
- `npx vitest run` (full suite, parallel — default config):
  ```
  Test Files  186 passed (186)
       Tests  1936 passed (1936)
  ```
- `npx vitest run --no-file-parallelism` (full suite, serial):
  ```
  Test Files  186 passed (186)
       Tests  1936 passed (1936)
  ```
  0 regressions in either mode. Baseline (AB4 closing state) was
  `1917/1917`; delta is `+19` new tests, all new, all passing (11 in
  `engram-backend.test.ts`, 1 async-dispatch test in
  `mcp-tools/tests/registry.test.ts`, 5 in `mcp-server/tests/server.test.ts`'s
  new "memory kind" describe block, 2 in `apps/cli/tests/mcp-serve.test.ts`).
- `npm run doctor` (self-dogfood, extra sanity):
  `41/54 OK · 0 FALLO · 3 ADVERTENCIA · 10 OMITIDO · Resultado: PASS` —
  identical to AB4's closing state.

## Result

`@axiom/memory` now has a REAL, LOCAL, cross-session backend
(`createEngramBackend`) that talks to engram's LOCAL MCP stdio server
(`engram mcp --project=<projectId>`), implementing the exact same
`MemoryBackend` interface as the original JSON backend — GATE 0024's
project-isolation guarantee is preserved and extended with a second,
process-level pin. `resolveMemoryBackend` provides automatic,
never-throwing fallback to the JSON backend when engram is not installed,
so every existing caller (tests, CI, users without engram) keeps working
unchanged. Topic-keyed UPSERT and session summaries were added
additively to both backends. `memory.decisionRecall`/`memory.
contextRecall` — previously declared capabilities with empty handler
arrays — are now wired end-to-end through a new `memory` MCP server kind,
with project-pinning enforced at the input-builder layer the same way
`INC-20260708-mcp-project-isolation-hardening` enforces it for `sdd.*`/
`spec.*`. Making `memory.*` genuinely asynchronous required promoting
`McpServer.handle()` to `async` project-wide — a mechanical, behavior-
preserving change confirmed by the full, otherwise-unmodified test suite
staying green (`1936/1936` in both parallel and serial runs). One real
infra gap was discovered and fixed during validation: `@axiom/providers`
(added in AB3) had never been linked into `node_modules/@axiom/`, which
only surfaced once `@axiom/memory` made it a transitive runtime
dependency of the compiled CLI `dist/` — fixed with `npm install`; the
existing E2E test (`workspace-mcp.e2e.test.ts`, which spawns the real
compiled binary) is what caught it, since ordinary vitest unit tests
resolve workspace packages via tsconfig path mapping and never exercise
real Node `require()` resolution.

## General spec integration

Integrated into `Axiom.Spec/specs/00-08` by the batch's final
consolidation pass (autopilot orchestrator). Landed in:

- **06_Integraciones_y_Capacidades.md** — PRIMARY. New subsection
  "Backend de memoria real (engram)": two `MemoryBackend`s +
  `resolveMemoryBackend`, engram MCP wire mapping, `memory.*` handlers +
  the `memory` `McpServerKind`, `McpServer.handle()` now async.
- **03_Modelo_Operativo_y_Datos.md** — memory data-model additions
  (`MemoryEntry.topicKey`/`.sessionId`, `MemorySessionSummary`,
  topic-keyed UPSERT on both backends) in the INC-20260708 data
  subsection.
- **01_Requisitos_Funcionales.md** — RF-AXM-026 (real persistent LOCAL
  cross-session memory).
- **07_Gobierno_y_Seguridad.md** — GATE 0024 preserved + extended with a
  process-level `--project=` pin; stricter `memory.*` input-builder
  pinning.
- **08_Glosario.md** — new terms: engram memory backend
  (`createEngramBackend`), topic-keyed memory / UPSERT.

The latent `node_modules/@axiom/providers` linkage gap this increment
fixed via `npm install` is an implementation detail, not stable spec
knowledge, so it stays in this README (not folded into 00-08).
