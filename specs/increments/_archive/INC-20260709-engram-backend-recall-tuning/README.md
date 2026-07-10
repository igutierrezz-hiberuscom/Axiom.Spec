# Increment: Engram backend recall tuning

Status: closed
Date: 2026-07-09

## Goal

Tune the Engram-backed `MemoryBackend` implementation
(`packages/memory/src/engram-backend.ts` in the `Axiom` repository) to
improve recall quality, without changing how the local `engram` process is
launched (its command, arguments, timeouts, or project pinning stay
untouched).

## Context

`createEngramBackend` wraps a local `engram mcp` process behind the same
`MemoryBackend` interface `createInMemoryBackend` implements. Two recall
weaknesses were identified by reading engram's own server source
(`C:\repos\engram\internal\store\store.go` and
`internal\mcp\mcp.go`):

1. Engram's `mem_search` tool accepts a `match_mode` argument: `"all"`
   (the default, FTS5 AND semantics — every query token must match) or
   `"any"` (FTS5 OR semantics — any query token matching is enough). The
   Axiom backend never set this argument, so it always used the stricter
   default, which under-recalls for multi-word queries.
2. Engram hard-clamps the `limit` argument to `MaxSearchResults = 20`
   server-side. The backend's `load()` method was requesting `limit: 50`,
   which engram silently truncates to 20 — the extra 30 was dead intent
   that never had any effect.

Additionally, `query()`'s existing kind filter (`query.kinds`) is applied
CLIENT-SIDE, after results already came back from engram (engram's
`mem_search` has no kind/type filter argument of its own). When a caller
asks for a small `limit` (e.g. 10) together with a kind filter, requesting
only that same small `limit` from engram risks the server's own unfiltered
ranking exhausting the requested count with entries of the WRONG kind,
starving the post-filter of candidates that would have matched.

## Scope

Changes are limited to `Axiom`'s `packages/memory` package:

- `packages/memory/src/engram-backend.ts`:
  - Introduce a module-level constant `ENGRAM_MAX_SEARCH_RESULTS = 20`,
    documented as mirroring engram's own `MaxSearchResults` clamp.
  - `load()`: request `limit: ENGRAM_MAX_SEARCH_RESULTS` (20) instead of
    the previous `limit: 50`.
  - `query()`: always pass `match_mode: 'any'` to `mem_search`, to widen
    recall for multi-word queries.
  - `query()`: when a kind filter (`query.kinds`) is active, request the
    full `ENGRAM_MAX_SEARCH_RESULTS` (20) from engram instead of the
    caller's own `limit`, giving the client-side kind filter the largest
    possible candidate pool. After filtering by kind, the result is
    sliced back down to the caller's originally requested `limit`
    (default 10), so callers still see the count they asked for.
- `packages/memory/tests/engram-backend.test.ts` and its stub fixture
  (`packages/memory/tests/fixtures/stub-engram-mcp-server.mjs`): extended
  with test cases covering the above, and a minimal, backward-compatible
  extension to the stub server so it records the exact `mem_search` args
  it received (for test assertions only — no change to the stub's
  response shape).

## Non-goals

- Do NOT change how `engram mcp` is launched: command, `--project=`/
  `--tools=agent` args, timeouts, working directory, or project-pinning
  logic are all unchanged.
- Do NOT change `save()` or `saveSessionSummary()`.
- Do NOT change `load()`'s project-guard (cross-project-blocked) logic —
  only its requested `limit` value changes.
- Do NOT introduce a `match_mode` option to `load()` — only `query()`
  gets `match_mode: 'any'` in this increment (the baked-in decision from
  the orchestrator scoped this narrowly; `load()`'s empty-query browse use
  case has no multi-token query to widen recall for).
- Do NOT touch the external `engram` or `GentleAI` repositories.

## Acceptance criteria

- [x] `query()` passes `match_mode: 'any'` to `mem_search`.
- [x] `load()` requests `limit: 20` (via the named
      `ENGRAM_MAX_SEARCH_RESULTS` constant), not 50.
- [x] With an active kind filter, `query()` requests `limit: 20` from the
      server and still honors the caller's own `limit` cap after
      filtering.
- [x] Build passes (`npm run build`, `tsc -b`, no errors).
- [x] Targeted tests pass (`npx vitest run packages/memory`); no
      regression in the pre-existing memory test suite.

## Open questions

None — decisions baked by orchestrator.

## Assumptions

- Engram's `MaxSearchResults` clamp (20) and `match_mode` vocabulary
  (`"all"` | `"any"`) are stable facts of the currently-installed local
  `engram` binary, confirmed directly against its Go source
  (`internal/store/store.go` ~L3104-3167, ~L3118-3119; default config
  ~L463/476) rather than inferred from behavior alone.
- No real project currently depends on `load()` returning more than 20
  entries in one call (it never could, even before this change — engram
  silently truncated any higher `limit` server-side).

## Implementation notes

`packages/memory/src/engram-backend.ts`:

- Added `const ENGRAM_MAX_SEARCH_RESULTS = 20` at module scope, with a
  comment citing engram's own `MaxSearchResults` server-side clamp as the
  reason requesting more is dead intent.
- `load()`'s `mem_search` call now sends `limit: ENGRAM_MAX_SEARCH_RESULTS`
  instead of the literal `50`.
- `query()` now computes:
  - `requestedLimit = query.limit ?? 10` (the caller's own intent).
  - `hasKindFilter = query.kinds !== undefined && query.kinds.length > 0`.
  - `searchLimit = hasKindFilter ? ENGRAM_MAX_SEARCH_RESULTS : requestedLimit`
    — the value actually sent to engram as `limit`.
  - The `mem_search` call always includes `match_mode: 'any'` alongside
    `limit: searchLimit`.
  - After building `entries` from the structured `results` array and
    applying the existing kind filter, entries are additionally sliced to
    `entries.slice(0, requestedLimit)` when a kind filter was active (the
    no-filter path already only ever received `requestedLimit` entries
    from the server, so no extra slice is needed there).

Test/fixture changes:

- `packages/memory/tests/fixtures/stub-engram-mcp-server.mjs`: the
  `mem_search` handler now records the raw `arguments` object it received
  (`query`, `project`, `limit`, `match_mode`, `scope`) into
  `state.lastMemSearchArgs`, persisted via the same
  `STUB_ENGRAM_STATE_FILE`-backed JSON file the stub already uses to
  survive across the backend's per-call process spawns. This is purely
  additive — no existing response envelope or behavior changed, so all
  pre-existing tests kept passing unmodified.
- `packages/memory/tests/engram-backend.test.ts`: added a
  `readLastMemSearchArgs()` helper that reads that state file, plus four
  new test cases:
  - `load()` requests `limit: 20`.
  - `query()` with no kind filter sends `match_mode: 'any'` and the
    caller's own `limit` (both an explicit `limit: 7` case and the
    default-to-`10` case).
  - `query()` with an active kind filter (6 same-kind matching entries,
    caller `limit: 3`) sends `limit: 20` to the server, yet the final
    returned array is still capped at 3.

## Validation

From `C:/repos/Axiom Workspace/Axiom`:

- `npm run build` (`tsc -b`) — clean, no errors.
- `npx vitest run packages/memory` — 4 test files, 55 tests, all passing
  (15 in `engram-backend.test.ts`, including the 4 new cases; no
  regression in the other 3 pre-existing memory test files).

No pre-existing failures apply to this targeted run (the known baseline
pre-existing failure, `packages/skills/tests/catalog.test.ts`, lives
outside `packages/memory` and was not part of this targeted validation
scope).

## Result

Implemented all three baked-in changes exactly as scoped: `match_mode:
'any'` on `query()`'s `mem_search` call, `load()`'s `limit` corrected from
the dead-intent `50` to the real server ceiling `20` via a named
`ENGRAM_MAX_SEARCH_RESULTS` constant, and `query()`'s kind-filtered path
now requests the full server ceiling before filtering, re-capping to the
caller's own requested limit afterwards. `launch()` and every other method
were left untouched. All acceptance criteria are satisfied and verified by
new, hermetic tests (stub MCP server fixture, no dependency on a real
`engram` binary) plus a clean build and full targeted test run.

## General spec integration

Integrada por el orchestrator en la pasada final cross-increment (batch
INC-20260709-*):

- `06_Integraciones_y_Capacidades.md` — nueva subsección "Tuning de recall
  del backend de memoria engram — INC-20260709-engram-backend-recall-tuning"
  (`match_mode:'any'` en `query()`, `ENGRAM_MAX_SEARCH_RESULTS=20` en
  `load()`, y pedir el tope 20 cuando hay filtro de kind client-side).
- `08_Glosario.md` — término "Tuning de recall de engram-backend" bajo el
  grupo "tanda INC-20260709-*".

Archivado en `Axiom.Spec/specs/increments/_archive/`.
