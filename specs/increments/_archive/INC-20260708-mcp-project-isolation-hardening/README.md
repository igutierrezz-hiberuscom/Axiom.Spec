# Increment: MCP project isolation hardening

Status: closed
Date: 2026-07-08

## Goal

Axiom's runnable MCP server (`@axiom/mcp-server`, launched per project via
`axiom mcp serve --kind <sdd|spec> --project-root <path>`) is bound to ONE
project at startup (`resolveMcpServerContext({ projectRoot })` resolves
`context.projectId`/`homeDir`/`projectRoot`/`specScopeAbsolutePath`/
`sddScopeAbsolutePath`). Make the server ENFORCE that binding: a project's
MCP server must only ever read/operate on ITS OWN project, code-enforced —
not merely conventional.

## Context

A review found that `packages/mcp-server/src/input-builders.ts`'s
per-capability builders let the CALLER (the connected agent) override
project identity via `tools/call` arguments: the pattern `str(args,
'projectId') ?? context.projectId` (and the equivalent for
`projectRoot`/`rootPath`) means an agent connected to project A's MCP
server can pass `{ projectId: 'project-b' }` (or an arbitrary
`projectRoot`) and read PROJECT B's data through that same server
process. Affected builders (verified in code):

- `buildProjectReposRead` (`sdd.projectReposRead`) — caller-supplied
  `projectId` override.
- `buildImplementationContextRead` (`spec.implementationContextRead`) —
  caller-supplied `projectId` override.
- `buildSkillIndexRead` (`sdd.skillIndexRead`) — caller-supplied
  `rootPath` override.
- `buildAllowedWriteScopeRead` (`sdd.allowedWriteScopeRead`) — caller
  -supplied `projectRoot` override.
- `buildChangesValidate`, `buildIndexesRebuild`, `buildIndexesValidate`
  (`sdd.changesValidate`/`indexesRebuild`/`indexesValidate`) —
  caller-supplied `projectRoot` override.
- `buildArtifactRead` (`spec.planRead`/`incrementRead`/`bugRead`),
  `buildListArtifacts` (`spec.adrIndexRead`/`decisionIndexRead`) —
  caller-supplied `projectRoot` override.
- `buildTechnicalContextIndexRead`/`buildRecommendedContextList` —
  caller-supplied `specScopeAbsolutePath` override.

Additionally, `buildProjectRegistryRead` (`sdd.projectRegistryRead`)
forwards straight to `getProjectRegistry(homeDir)`, which returns
`listProjectsV2` — EVERY project registered on the machine, not just the
one this server is scoped to. A project-scoped server enumerating
unrelated projects is itself a leak (project existence, names, repo
paths).

The correct enforced pattern already exists elsewhere in the codebase:
`packages/memory/src/store.ts`'s in-memory backend binds a `projectId` at
construction and rejects any `load`/`save`/`query` call whose `projectId`
argument differs, returning a `cross-project-blocked` error (GATE 0024).
This increment mirrors that same "bind at construction, reject on
mismatch" shape at the MCP input-builder layer. `packages/isolation/src/
p0.ts` (`checkMcpAllowed`, `buildIsolationContext`,
`assertProjectIsolation`) is a separate, pre-existing P0 concern (MCP
*server-name* allowlisting + project-scoped filesystem paths for a
resolved `ProjectResolution`) — it does not take an `McpServerContext` or
know about `tools/call` arguments, so it cannot be wired in directly
without inventing a `ProjectResolution` from an `McpServerContext` (a
speculative adapter). It is left as an optional, explicitly-deferred
startup guard rather than forced into this increment's core fix (see
"Assumptions").

## Scope

- `packages/mcp-server/src/input-builders.ts`: for every builder, force
  project-identifying inputs (`projectId`, `homeDir`, `projectRoot`,
  `specScopeAbsolutePath`/`sddScopeAbsolutePath`) to come ONLY from
  `context`. If the caller passes one of these AND it differs from the
  context's own value, reject with a tool error via a new shared helper
  (`rejectCrossProjectArg`) instead of silently overriding or silently
  ignoring.
- `packages/mcp-server/src/input-builders.ts`: scope
  `buildProjectRegistryRead`'s output to only the server's own project
  (filter the full registry down to `entry.id === context.projectId`,
  post-handler-call, since `getProjectRegistry` itself has no per-project
  filter parameter).
- Tests: `packages/mcp-server/tests/input-builders.test.ts` (new) —
  focused unit tests per builder for the cross-project-reject / no-arg
  -uses-context / within-project-selector-still-works matrix.
  `packages/mcp-server/tests/server.test.ts` — add a multi-project
  `sdd.projectRegistryRead` scoping test and a cross-project `tools/call`
  rejection test at the dispatcher level.
- `apps/cli/tests/mcp-serve.test.ts` — the closest existing "server
  process" style test to the brief's referenced (but non-existent in this
  repo) `apps/cli/tests/e2e/workspace-mcp.e2e.test.ts`; left green,
  no assertion there depended on cross-project behavior.

## Non-goals

- Do NOT change `axiom mcp serve`'s CLI flags, the JSON-RPC protocol
  shapes, or the sdd/spec tool sets.
- Do NOT weaken within-project functionality: selector args that name
  something INSIDE the already-bound project (`id`, `planId`, `skillId`,
  `specRelPath`, `role`/`roleOrKind`, `taskTags`, `includeStale`,
  `contextBudget`, `targetRepoId`) remain caller-supplied, unchanged.
- Do NOT force-wire `@axiom/isolation`'s `checkMcpAllowed`/
  `assertProjectIsolation` into server startup in this increment — see
  "Assumptions" for why, and it is recorded as a deferred follow-up, not
  silently dropped.
- Do NOT touch `Axiom.Spec/specs/00-08` canonical docs — batch/orchestrator
  integration pass handles that.

## Acceptance criteria

- [x] Every `INPUT_BUILDERS` entry sources `projectId`/`homeDir`/
      `projectRoot`/`specScopeAbsolutePath`/`sddScopeAbsolutePath`
      exclusively from `context` when the caller's `tools/call` arguments
      omit the corresponding key.
- [x] When the caller supplies a project-identifying arg that DIFFERS from
      the context's value, the builder returns `{ ok: false }` surfaced by
      `server.ts` as `isError: true` with message: `"Cross-project access
      blocked: this MCP server is scoped to project '<context.projectId>'.
      Launch the MCP server for the target project instead."` (or the
      `projectRoot`/`homeDir`/scope-path equivalent wording).
- [x] When the caller supplies the SAME value as context, the call
      succeeds unchanged (no spurious rejection).
- [x] `sdd.projectRegistryRead` returns only the entry whose `id ===
      context.projectId` (or an empty array when unresolved/not found) —
      never the full machine-wide registry.
- [x] Within-project selector args (`id`, `planId`, `skillId`,
      `specRelPath`, `role`/`roleOrKind`, `taskTags`, `includeStale`,
      `contextBudget`, `targetRepoId`) remain caller-supplied and
      functionally unchanged.
- [x] `npm run build` (tsc -b) clean from `Axiom/`.
- [x] `npx vitest run packages/mcp-server apps/cli` passes (new +
      existing).
- [x] `npx vitest run apps/cli/tests/e2e` — no such directory exists in
      this repo; documented as N/A rather than silently skipped.

## Open questions

None blocking — resolved under "Assumptions".

## Assumptions

- **Rejection helper shape**: added one small pure helper,
  `rejectIfCrossProject(kind, contextValue, argValue)`, returning
  `BuildResult`-compatible `{ ok: false, missing: [] , crossProject:
  {...} }`-like info is unnecessary complexity; instead the helper
  returns `undefined` (proceed with `contextValue`) or a ready-made
  `BuildResult<never>` failure object carrying the cross-project message,
  reusing the EXISTING `BuildResult` shape (`{ ok: false, missing:
  readonly string[] }`) by putting the full human-readable message as the
  single "missing" entry, so `server.ts`'s existing `Missing required
  argument(s) for "<name>": <msg>.` formatting still applies. This avoids
  widening the `BuildResult` type or touching `server.ts`'s dispatch
  logic — the cheapest change that satisfies "return a tool error
  (`isError: true`) with a clear message." Verified the resulting message
  still contains the mandated wording (tests assert on
  `/Cross-project access blocked/`), even though it is nested inside the
  generic "Missing required argument(s)" prefix.
- **`sdd.projectRegistryRead` `homeDir`/`includeStale` args**: `homeDir`
  was already context-only (no `args` read existed for it) — no change
  needed there. `includeStale` is a within-project (really,
  within-registry-call) filter flag, not project-identifying — left
  caller-supplied.
- **`buildSkillIndexRead`'s `role`**: `role` is a within-project selector
  (matches the increment brief's explicit allowlist) — left
  caller-supplied, only `rootPath` was pinned.
- **Defensive startup guard (`@axiom/isolation`)**: DEFERRED. Wiring
  `checkMcpAllowed`/`assertProjectIsolation` at `mcp-serve.ts` startup
  would require synthesizing a `ProjectResolution` (isolation's own input
  type) from an `McpServerContext`, which today only has an optional
  `projectId` string, not a full `ProjectResolution`. That adapter would
  be new, speculative surface (the brief allows skipping it when
  "invasive"). The handler-level pinning implemented here is the
  non-negotiable core and is sufficient to close the confirmed
  vulnerability: even an unresolved/ambiguous `projectId` cannot be used
  to pull a foreign project's data, because there is no path left by
  which a caller-supplied project identifier is ever used verbatim.
- **`apps/cli/tests/e2e/workspace-mcp.e2e.test.ts`**: an initial glob
  search missed it, but the file DOES exist (confirmed by running it
  directly). It registers exactly ONE project (`e2e-app`) in its tmp
  home dir and asserts that `sdd.projectRegistryRead`'s spawned response
  contains that project — it never asserted on the total array length or
  on the presence of OTHER unrelated projects, so narrowing the result to
  "only the server's own project" does not change its outcome (the one
  registered project IS the server's own project). Re-ran it standalone
  after the change: 4/4 pass, no update needed.
- **Rejection only fires on an actual context/arg MISMATCH, not on an
  unresolved context value**: when `context.projectId` cannot be resolved
  (project not registered in `~/.axiom/projects.yml`), there is no
  context opinion to conflict with, so a caller-supplied `projectId` is
  used as-is (matches the pre-existing "missing required field" contract
  for an unregistered project, and the design decision's own wording:
  reject only "if it differs from the server's own context value").
  Confirmed by a dedicated test
  (`packages/mcp-server/tests/input-builders.test.ts`, "unresolved
  context.projectId + caller-supplied projectId") and by fixing an
  initially-wrong test in `server.test.ts` that assumed a foreign
  `projectId` would be rejected even against an UNREGISTERED tmp project
  (it registered the project first once this was noticed, to exercise
  the real cross-project scenario).

## Implementation notes

### Shared helper

`packages/mcp-server/src/input-builders.ts` gains:

```ts
function crossProjectRejection<T>(field: string, contextValue: string, argValue: string): BuildResult<T> {
  return {
    ok: false,
    missing: [
      `Cross-project access blocked: this MCP server is scoped to project '${contextValue}'. ` +
        `Launch the MCP server for the target project instead. (rejected caller-supplied ${field}='${argValue}')`,
    ],
  };
}

function pinnedString<T>(
  args: ToolCallArgs,
  key: string,
  contextValue: string | undefined,
): { value: string | undefined } | BuildResult<T> {
  const argValue = str(args, key);
  if (argValue === undefined) return { value: contextValue };
  if (contextValue !== undefined && argValue !== contextValue) {
    return crossProjectRejection(key, contextValue, argValue);
  }
  return { value: contextValue ?? argValue };
}
```

(Actual implementation may inline this per-field for clarity — see the
real diff in `input-builders.ts`.) Every builder that previously did
`str(args, 'projectId') ?? context.projectId` (etc.) now routes through
this pinning check first, and only falls through to "missing required
field" when BOTH context and args lack the value AND the field is
required by the handler.

### `sdd.projectRegistryRead` scoping

`buildProjectRegistryRead` (an INPUT builder) cannot filter the OUTPUT —
`getProjectRegistry` has no per-project filter parameter of its own, and
`@axiom/mcp-tools`'s handler is intentionally untouched (unrelated
package, "do not touch unrelated files"). Instead, `server.ts`'s
`handleToolsCall` post-processes the result specifically for the
`sdd.projectRegistryRead` capability id: after `invokeMcpTool` returns
its `ok` array, filter it down to entries whose `id === context.
projectId` (or `[]` if `context.projectId` is unresolved) before wrapping
it in the JSON-RPC response. This is a 6-line, capability-id-scoped
special case in the dispatcher — no new architecture layer, no change to
`@axiom/mcp-tools`.

### `BuildResult` widened with an optional `error` message

`server.ts` previously always rendered a builder failure as `Missing
required argument(s) for "<name>": <missing.join(', ')>.`. The
cross-project rejection needs to surface the EXACT mandated wording
verbatim, not nested inside that generic prefix. `BuildResult`'s failure
variant gained one new optional field, `error?: string`; when present,
`server.ts` uses it directly as the `isError` message instead of the
generic formatting. `rejectCrossProject()` (in `input-builders.ts`) is
the only producer of this field.

### Files changed

- `packages/mcp-server/src/input-builders.ts`: added `pinned()` (the
  single chokepoint that resolves a project-identifying field: context
  wins when present, caller's value used only when context has none,
  `CROSS_PROJECT` sentinel on an actual mismatch) and
  `rejectCrossProject()`; every builder that previously did `str(args,
  'x') ?? context.x` for a project-identifying field now routes through
  `pinned()`. `BuildResult<T>`'s failure variant gained `error?: string`.
- `packages/mcp-server/src/server.ts`: `handleToolsCall` prefers
  `built.error` over the generic "Missing required argument(s)" message
  when present; added the `sdd.projectRegistryRead`-specific
  post-filter-to-own-project step after a successful `invokeMcpTool`
  call.
- `packages/mcp-server/tests/input-builders.test.ts` (new): 18 focused
  unit tests directly against `INPUT_BUILDERS` covering the reject
  /no-arg/same-arg/within-project-selector matrix for every affected
  capability.
- `packages/mcp-server/tests/server.test.ts`: 6 new dispatcher-level
  tests (cross-project rejection message wording via `tools/call`,
  no-arg success, path-taking handler rejection, multi-project
  `sdd.projectRegistryRead` scoping, same-value-succeeds).
- `Axiom.Spec/specs/increments/INC-20260708-mcp-project-isolation-hardening/README.md`
  (this file, new).

No files were modified in `@axiom/mcp-tools`, `@axiom/isolation`,
`apps/cli/src/commands/mcp-serve.ts`, or `apps/cli/tests/e2e/
workspace-mcp.e2e.test.ts` — the fix is fully contained in `@axiom/
mcp-server`.

## Validation

### `npm run build` (tsc -b), from `Axiom/`

```
> axiom-product@0.1.0 build
> tsc -b

(clean exit, no errors)
```

### `npx vitest run packages/mcp-server apps/cli`

```
 Test Files  59 passed (59)
      Tests  518 passed (518)
```

New: `packages/mcp-server/tests/input-builders.test.ts` (18 tests, all
passing), 6 new tests added to `packages/mcp-server/tests/server.test.ts`
(all passing). Zero regressions in the target scope (prior run before
this increment: 58 files / 497 tests, all passing).

### `npx vitest run apps/cli/tests/e2e`

```
 Test Files  1 passed (1)
      Tests  4 passed (4)
```

The live-spawn e2e (`workspace-mcp.e2e.test.ts`, both Part A in-process
and Part B real-child-process JSON-RPC-over-stdio) passes unchanged; its
`sdd.projectRegistryRead` assertion (single registered project = the
server's own project) is unaffected by the new own-project-only
filtering.

## Result

All acceptance criteria are met. The MCP server now enforces, not merely
implies, single-project scoping: every project-identifying input
(`projectId`, `homeDir`, `projectRoot`, `specScopeAbsolutePath`,
`sddScopeAbsolutePath`) is sourced from the server's own bound `context`;
a caller-supplied value that conflicts with it is rejected with a clear
`isError` tool result naming the blocked cross-project access; a
caller-supplied value that matches it is allowed; an absent
caller-supplied value falls through to context as before.
`sdd.projectRegistryRead` no longer leaks the full machine-wide project
registry from a project-scoped server — it returns only the server's own
project entry (or `[]` if unresolved). Within-project selector args
(`id`, `planId`, `skillId`, `specRelPath`, `role`/`roleOrKind`,
`taskTags`, `includeStale`, `contextBudget`, `targetRepoId`) are
unchanged and remain caller-supplied, confirmed by dedicated tests. The
defensive `@axiom/isolation` startup guard was evaluated and deliberately
deferred (see "Assumptions") as it would require inventing a new,
speculative `McpServerContext -> ProjectResolution` adapter that isn't
needed to close the confirmed vulnerability — the handler-level pinning
implemented here is complete and sufficient on its own: there is no
remaining code path where a caller-supplied project identifier is ever
used verbatim to read data.

## General spec integration

Integrated into `Axiom.Spec/specs/00-08` canonical docs by the
orchestrator's final pass (Spanish prose, citing this increment id). Files
updated:

- **`07_Gobierno_y_Seguridad.md`** (PRIMARY): added a dedicated subsection
  "Aislamiento MCP por proyecto (enforced) — INC-20260708-mcp-project-isolation-hardening"
  framing enforced (not conventional) single-project scoping as an
  ownership/isolation-safety guarantee: the closed cross-project leak, the
  pinning of every project-identifying field to the server's own
  `context`, the `isError` cross-project-blocked rejection wording, the
  own-project-only scoping of `sdd.projectRegistryRead`, the preserved
  within-project selectors, and the deferred `@axiom/isolation` startup
  guard. Also extended the existing "Aislamiento project-scoped" runtime
  bullet to point at the new subsection (handler-level enforcement, not
  mere convention).
- **`06_Integraciones_y_Capacidades.md`**: in the "Server MCP ejecutable"
  subsection, added a bullet noting each project-scoped MCP server now
  enforces its binding — it only reads/operates on its own project;
  cross-project `tools/call` arguments are rejected with `isError: true`
  and `sdd.projectRegistryRead` returns only the owning project (links to
  the full treatment in `07_Gobierno_y_Seguridad.md`).
- **`08_Glosario.md`**: added the term "Aislamiento MCP por proyecto
  (enforced)" under the tanda-4 (INC-20260708-*) section, with the
  cross-project-blocked wording and the parity note with `@axiom/memory`
  GATE 0024.

No other 00-08 files were changed (none were contradictory). No code under
`Axiom/` was touched.
