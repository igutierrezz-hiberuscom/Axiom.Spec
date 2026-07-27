# Increment: Adapter MCP parity — vscode/visual-studio-2026 native writers + codex/antigravity informative notes

Status: closed
Date: 2026-07-26

## Goal

Extend native MCP config projection (`writeNativeMcpConfig`,
`apps/cli/src/commands/native-mcp-config.ts`) so every canonical adapter
target with a verified or explicitly documented-assumption native MCP
schema gets a real, merge-preserving MCP config file; and every remaining
target gets a clear, actionable informative note instead — NEVER an
invented, unverified native config schema. Concretely:

- `vscode` (VERIFIED): routes to VS Code's own native MCP file,
  `.vscode/mcp.json` — the SAME file/schema `copilot-vscode`/
  `github-copilot` already write to.
- `visual-studio-2026` (DOCUMENTED ASSUMPTION, override-able, NOT
  independently re-verified): a NEW `.vs/mcp.json`, same
  `{ servers: { <id>: { type:'stdio', ... } } }` shape as VS Code, at its
  OWN path to avoid colliding with claude-code's root `.mcp.json`
  (`{ mcpServers }`, a different key).
- `codex`/`antigravity`: their real MCP config is USER-GLOBAL, not
  project-scoped (`~/.codex/config.toml`'s `[mcp_servers]`;
  `~/.gemini/config/mcp_config.json`'s `mcpServers`) — Axiom never writes
  a project file for them; instead `writeNativeMcpConfig` returns an
  explicit, informative warning naming the exact global location + the
  launchable Axiom servers (`sdd-mcp-server`/`spec-mcp-broker`, with their
  real `command`/`args`) the user should add there manually.
- Confirm the two real production callers of `writeNativeMcpConfig`
  (`writeWorkspaceNativeMcpConfigs` in `workspace-mcp.ts`,
  `provisionWorktreeExecution` in `workspace-worktree-provision.ts`)
  actually dispatch to the new/extended `case`s rather than silently
  degrading to their own generic fallback message.
- Confirm the portable `.axiom/skills/` + `.axiom/agents/` surface
  (`materializeProcessSurfaces`, `workspace-process-surfaces.ts`) is
  adapter-agnostic/unconditional, so codex/antigravity/vscode/vs2026 are
  already discoverable through it without needing a native per-IDE skill
  format in this increment.

This is INC-3 of the 3-increment `codex`/`antigravity`/`visual-studio-2026`
parity track: INC-1 (`INC-20260726-adapter-registry-canonical`) registered
`codex`/`vscode` as recognized adapter target ids; INC-2
(`INC-20260726-adapter-generators`) gave `codex`/`antigravity`/
`visual-studio-2026` dedicated `@axiom/adapters-<target>` AGENTS.md/AXIOM.md
generator packages, explicitly deferring "native MCP config projection...
INC-3, not here". This increment is that INC-3.

## Context

Before this increment, `NATIVE_MCP_TARGETS` (the gate deciding which
adapter targets get a project-scoped native MCP file) was exactly 5 ids:
`claude-code`, `cursor`, `copilot-vscode`, `github-copilot`, `opencode`.
`vscode` (a real, distinct, already-registered adapter target since INC-1,
with its own AGENTS.md-equivalent generator package) had NO native MCP
projection at all — selecting it wrote zero MCP config, silently. `codex`/
`antigravity`/`visual-studio-2026` all fell through `writeNativeMcpConfig`'s
non-exhaustive `default:` branch, receiving the SAME generic message
("Native MCP config projection not available for target 'X' yet") with no
distinction between "genuinely no schema exists" (`litellm`) and "the real
schema is user-global, here's exactly where to add it" (`codex`/
`antigravity`).

`materializeProcessSurfaces` (NS-2, `workspace-process-surfaces.ts`) already
materializes a portable, adapter-agnostic `.axiom/{agents,commands,skills}/`
surface for every repo regardless of which native adapters are selected —
confirmed by direct source inspection (see Implementation notes) rather
than assumed.

## Scope

- `apps/cli/src/commands/native-mcp-config.ts`:
  - `case 'vscode':` added to the existing `copilot-vscode`/`github-copilot`
    case block (same `writeVscodeMcpConfig` call, same
    `VSCODE_NATIVE_MCP_RELATIVE_PATH`). `'vscode'` added to
    `NATIVE_MCP_TARGETS`.
  - New exported constant `VS2026_NATIVE_MCP_RELATIVE_PATH =
    path.join('.vs', 'mcp.json')` + `case 'visual-studio-2026':` routing to
    `writeVscodeMcpConfig` at that path (same shape as VS Code, different
    file). `'visual-studio-2026'` added to `NATIVE_MCP_TARGETS`.
  - New `case 'codex':` / `case 'antigravity':` — each returns a single
    informative warning (via the new `buildUserGlobalMcpNote` helper),
    `path: undefined`, and writes nothing. Both remain OUT of
    `NATIVE_MCP_TARGETS` (that set gates "does a project file get
    written?"). New exported constant `NATIVE_MCP_INFORMATIVE_TARGETS =
    ['codex', 'antigravity']` — a second, distinct gate for "has an
    explicit, no-file `case`" that the two real callers below now also
    check.
  - Header doc comment (targets table) and `NATIVE_MCP_TARGETS`'s own doc
    comment rewritten to describe all 7 project-scoped targets, the 2
    user-global informative targets, and `litellm` as the sole remaining
    "no schema, no case" target.
- `apps/cli/src/commands/workspace-mcp.ts` (`writeWorkspaceNativeMcpConfigs`):
  gating changed from `NATIVE_MCP_TARGETS.includes(a)` alone to
  `NATIVE_MCP_TARGETS.includes(a) || NATIVE_MCP_INFORMATIVE_TARGETS.includes(a)`
  — so `codex`/`antigravity` are dispatched through `writeNativeMcpConfig`
  (surfacing their informative note) instead of falling into the bulk
  `unsupportedAdapters` generic message.
- `apps/cli/src/commands/workspace-worktree-provision.ts`
  (`isNativeMcpTarget`): same gating fix (its own private guard, used to
  decide whether to call `writeNativeMcpConfig` at all vs. this file's own
  "No verified native MCP config schema" message) — extended to also
  accept `NATIVE_MCP_INFORMATIVE_TARGETS`. Found during this increment's
  own investigation (a second, independent gate with the identical
  pre-existing gap the brief named for `workspace-mcp.ts`); fixed for the
  same reason, kept minimal (one guard function).
- Tests: `apps/cli/tests/native-mcp-config.test.ts` (vscode/
  visual-studio-2026/codex/antigravity cases, `NATIVE_MCP_TARGETS`'s new
  7-target list), `apps/cli/tests/workspace-mcp.test.ts` (vscode +
  visual-studio-2026 both written from `writeWorkspaceNativeMcpConfigs`;
  codex/antigravity write no file but surface their informative warning),
  `apps/cli/tests/workspace-worktree-provision.test.ts` (codex/antigravity
  route through the informative `case` instead of this file's own generic
  message), `apps/cli/tests/workspace-process-surfaces.test.ts` (one new
  test proving the portable surface is written even with `adapters: []`).

## Non-goals

- Native per-IDE skill/agent file formats for `cursor`/`copilot-vscode`/
  `github-copilot`/etc. — explicitly DEFERRED. Skills/agents are already
  discoverable through the portable `.axiom/{agents,commands,skills}/`
  surface (unconditional, confirmed in this increment) plus each adapter's
  own AGENTS.md/AXIOM.md pointer to it; native per-command files are a
  future refinement, not a blocker for MCP reachability.
- Re-verifying Visual Studio 2026's real MCP schema/location
  independently — `.vs/mcp.json` + the stdio shape stays a DOCUMENTED,
  override-able ASSUMPTION (same discipline already applied to
  cmm/serena binary-name assumptions elsewhere in this codebase), not a
  confirmed fact.
- Writing ANY project-scoped file for `codex`/`antigravity` — their real
  MCP config is user-global; inventing a project file they wouldn't even
  read would violate the "never invent an unverified schema" discipline
  in a different direction (inventing a LOCATION, not just a shape).
- Touching `litellm`'s handling — still no schema, still the fully
  generic `default:` fallback message, unchanged.
- Updating `packages/adapters/visual-studio-2026`'s/`antigravity`'s own
  bundled AGENTS.md/AXIOM.md adapter-note text (`VISUAL_STUDIO_2026_ADAPTER_NOTE`,
  `ANTIGRAVITY_ADAPTER_NOTE` in their respective `agents-md.ts`), which
  still literally say "no existe hoy un schema nativo de MCP verificado" /
  "la proyección de MCP nativo verificada llega en un incremento
  posterior" — now stale for `visual-studio-2026` (it has a
  documented-assumption schema as of this increment) and for the general
  "later increment" framing (this increment IS that later increment for
  all 3 targets). Explicitly OUT of this increment's file scope (that
  package belongs to INC-2, already closed); flagged here as a known,
  small follow-up rather than silently left inconsistent.

## Acceptance criteria

- [x] `vscode` writes `.vscode/mcp.json` (`{ servers: { <id>: { type:'stdio',
      ... } } }`), same file/shape as `copilot-vscode`/`github-copilot`.
- [x] `visual-studio-2026` writes `.vs/mcp.json`, same `servers`/
      `type:'stdio'` shape as VS Code, at a DIFFERENT path than both
      `.vscode/mcp.json` and claude-code's `.mcp.json` — proven never to
      collide when both `claude-code` and `visual-studio-2026` are
      selected together.
- [x] `codex`/`antigravity` write NO file (proven: `fs.readdirSync(repoRoot)`
      empty after the call) and return exactly one warning naming the
      real user-global location (`~/.codex/config.toml` / `~/.gemini/
      config/mcp_config.json`), the expected key
      (`[mcp_servers]`/`mcpServers`), and the launchable Axiom servers
      (`sdd-mcp-server`/`spec-mcp-broker`, with real `command`/`args`).
- [x] `NATIVE_MCP_TARGETS` (project-file gate) is exactly the 7 ids:
      `claude-code`, `cursor`, `copilot-vscode`, `github-copilot`,
      `opencode`, `vscode`, `visual-studio-2026` — `codex`/`antigravity`
      explicitly excluded.
- [x] Both real production callers of `writeNativeMcpConfig`
      (`writeWorkspaceNativeMcpConfigs`, `provisionWorktreeExecution`)
      dispatch `vscode`/`visual-studio-2026` to a real file write and
      `codex`/`antigravity` to the informative note — proven with
      dedicated tests in each of the 3 affected test files, not just at
      the `native-mcp-config.ts` unit level.
- [x] The portable `.axiom/{agents,commands,skills}/` surface materializes
      unconditionally (even with `adapters: []`) — proven with a
      dedicated test.
- [x] `npm run build` (`tsc -b`) passes clean.
- [x] All targeted test files pass; `packages/doctor` shows zero
      regression (confirmed it has no references to `NATIVE_MCP_TARGETS`/
      native-mcp-config at all, so nothing there could regress); the full
      `apps/cli` suite passes with zero failures.

## Open questions

None blocking.

## Assumptions

1. **`visual-studio-2026`'s `.vs/mcp.json` + `{ servers: { <id>: {
   type:'stdio', ... } } }` shape is a DOCUMENTED ASSUMPTION, explicitly
   NOT independently re-verified against Visual Studio's real MCP
   support.** Visual Studio 2022 17.14+/2026 is documented to read MCP
   config from solution-level locations; this increment assumes
   `.vs/mcp.json` using the SAME shape VS Code uses (a reasonable,
   override-able default given no more specific documentation was
   available), rather than inventing a wholly distinct, made-up schema.
   `.vs/mcp.json` (not root `.mcp.json`) was chosen specifically so it
   never collides with claude-code's `.mcp.json` (`{ mcpServers }`, a
   DIFFERENT key) when both targets are selected in the same repo — proven
   by a dedicated test. If Visual Studio's real schema/location turns out
   to differ, only `VS2026_NATIVE_MCP_RELATIVE_PATH` + its one `case`
   need to change; nothing else in `writeNativeMcpConfig`'s contract.
2. **`codex`/`antigravity` needed a SECOND exported gate,
   `NATIVE_MCP_INFORMATIVE_TARGETS`, distinct from `NATIVE_MCP_TARGETS`.**
   The brief's file-scope for the codex/antigravity `case`s was
   `native-mcp-config.ts` only; but both real production callers
   (`writeWorkspaceNativeMcpConfigs` in `workspace-mcp.ts`,
   `provisionWorktreeExecution`'s `isNativeMcpTarget` guard in
   `workspace-worktree-provision.ts`) pre-filter which targets ever reach
   `writeNativeMcpConfig` using `NATIVE_MCP_TARGETS.includes(...)` alone —
   and `codex`/`antigravity` are deliberately NOT in that set (it gates
   "gets a project file", not "has explicit handling"). Without a second
   gate, the new informative `case`s would be dead code from the real
   CLI's perspective: both callers would keep emitting their OWN generic
   "not available"/"no verified native MCP config schema" message instead
   of ever invoking the new, more helpful `case`. `NATIVE_MCP_INFORMATIVE_TARGETS`
   (`['codex', 'antigravity']`) closes that gap with minimal, focused
   edits to both callers' existing gating expressions — no other logic in
   either file changed.
3. **`workspace-worktree-provision.ts` was touched even though the brief's
   Part B named only `workspace-mcp.ts` and `workspace-process-surfaces.ts`
   explicitly.** It is a second, independent real caller of
   `writeNativeMcpConfig` with the IDENTICAL pre-existing gating pattern
   the brief asked to audit in `workspace-mcp.ts`; leaving it unfixed would
   have meant `axiom-role start --worktree` with `codex`/`antigravity`
   selected still degraded to the old, less-informative message — directly
   contradicting this increment's stated goal ("so their agents can
   actually launch/reach Axiom's `sdd-mcp-server`/`spec-mcp-broker`", which
   for these two targets specifically means "know exactly where to add
   them manually"). The fix is a 3-line guard extension (`isNativeMcpTarget`
   now also accepts `NATIVE_MCP_INFORMATIVE_TARGETS`), confirmed safe by
   grepping this repo's tests for any assertion on the literal string "No
   verified native MCP config schema" or on `codex`/`antigravity` in this
   file's test suite (none existed), and covered by a new, dedicated
   parameterized test.
4. **`packages/adapters/visual-studio-2026`'s/`antigravity`'s own bundled
   AGENTS.md/AXIOM.md note text is NOT updated in this increment** (see
   Non-goals) — that package's content belongs to INC-2's file scope
   (`packages/adapters/{visual-studio-2026,antigravity}/src/agents-md.ts`),
   already closed; touching it here would violate "keep changes small and
   focused" for a cosmetic/documentation-only inconsistency, not a
   functional one (the REAL MCP wiring this increment is about doesn't
   read that note text at all). Flagged explicitly as a small follow-up
   rather than silently left stale.

## Implementation notes

### `native-mcp-config.ts` — final dispatch table

| Target | File written | Shape | Verification |
|---|---|---|---|
| `claude-code` | `.mcp.json` | `{ mcpServers }` | VERIFIED |
| `cursor` | `.cursor/mcp.json` | `{ mcpServers }` | VERIFIED |
| `copilot-vscode` / `github-copilot` / `vscode` | `.vscode/mcp.json` | `{ servers, type:'stdio' }` | VERIFIED |
| `opencode` | `opencode.json` | `{ mcp, command:[...], type:'local', enabled }` | VERIFIED |
| `visual-studio-2026` | `.vs/mcp.json` | `{ servers, type:'stdio' }` | DOCUMENTED ASSUMPTION |
| `codex` | none | — (note: `~/.codex/config.toml`, `[mcp_servers]`) | user-global |
| `antigravity` | none | — (note: `~/.gemini/config/mcp_config.json`, `mcpServers`) | user-global |
| `litellm` (and any other future id) | none | — (generic "not available" warning) | no schema |

`NATIVE_MCP_TARGETS` = the first 5 rows (7 ids, since `vscode` shares a row
with `copilot-vscode`/`github-copilot`). `NATIVE_MCP_INFORMATIVE_TARGETS` =
`['codex', 'antigravity']`. `litellm` is in neither set.

### `buildUserGlobalMcpNote`

New private helper in `native-mcp-config.ts`: given a target name, a
user-global location string, an expected key hint, and the same
`NativeMcpServerInput[]` every other writer receives, it composes a single
warning string naming the location + key + each launchable server's real
`id`/`command`/`args` (falling back to a generic "sdd-mcp-server /
spec-mcp-broker" phrase only if, degenerately, no server has a `command`
at all). Used identically by both the `codex` and `antigravity` `case`s,
parameterized only by location/key.

### Portable process-surfaces unconditionality (confirmed, not assumed)

`materializeProcessSurfaces` (`workspace-process-surfaces.ts`) writes
`.axiom/agents/<id>.md`, `.axiom/commands/<id>.md`, and
`.axiom/skills/<id>/SKILL.md` for every applicable surface definition
UNCONDITIONALLY, before any adapter-specific branch (`.claude/...` gated on
`adapters.has('claude-code')`, `.opencode/...` gated on
`adapters.has('opencode')`, etc.) — see the loop at
`apps/cli/src/commands/workspace-process-surfaces.ts:972-990`, which runs
for every `def` regardless of `args.adapters`. A new test
(`workspace-process-surfaces.test.ts`) proves this directly by calling
`materializeProcessSurfaces({ ..., adapters: [] })` and asserting the 3
portable files still exist while no adapter-specific directory
(`.claude`, `.opencode`) is created. No source change was needed here —
this item was pure confirmation, as anticipated by the brief.

## Validation

Ran from `C:\repos\Axiom Workspace\Axiom`:

- `npm run build` (`tsc -b`) — PASSED, clean, exit 0.
- `npx vitest run apps/cli/tests/native-mcp-config.test.ts
  apps/cli/tests/workspace-mcp.test.ts
  apps/cli/tests/workspace-worktree-provision.test.ts
  apps/cli/tests/workspace-process-surfaces.test.ts` — **4 files / 72
  tests passed** (21 in `native-mcp-config.test.ts`, up from the
  pre-existing count, covering `vscode`/`visual-studio-2026`/`codex`/
  `antigravity`/`litellm`; 33 in `workspace-mcp.test.ts`, including the 2
  new dedicated cases; 12 in `workspace-worktree-provision.test.ts`,
  including the new codex/antigravity parameterized case; 6 in
  `workspace-process-surfaces.test.ts`, including the new unconditional-
  portable-surface case).
- `npx vitest run apps/cli/tests/e2e/adapters.e2e.test.ts
  apps/cli/tests/e2e/cross-cutting-batch.e2e.test.ts
  apps/cli/tests/e2e/workspace-mcp.e2e.test.ts` (every other file the
  brief's own grep for `writeNativeMcpConfig`/`workspace-mcp`/
  `NATIVE_MCP` turned up) — **3 files / 11 tests passed**, zero
  regressions (none of these files asserted anything specific to
  `codex`/`antigravity`/`vscode`/`visual-studio-2026`'s native MCP
  behavior, so no test edits were needed there).
- `npx vitest run packages/doctor` — **18 files / 182 tests passed**.
  Confirmed via grep beforehand that no file under `packages/doctor`
  references `NATIVE_MCP_TARGETS` or `native-mcp-config` at all, so this
  was a pure no-regression safety pass, not an expected-change area.
- `npx vitest run apps/cli/tests` (full suite, extra safety pass) — **125
  files / 1167 tests passed**, zero failures.

No pre-existing failures were encountered in any touched or adjacent
scope.

## Result

Implemented. `vscode` and `visual-studio-2026` now materialize real,
merge-preserving native MCP config files (`.vscode/mcp.json` and the new
`.vs/mcp.json` respectively, both `{ servers, type:'stdio' }`) through both
real production call sites (`writeWorkspaceNativeMcpConfigs` via `axiom
workspace setup`/`runWorkspaceSetup`, and `provisionWorktreeExecution` via
worktree-mode role starts). `codex`/`antigravity` never get an invented
project file — instead, both call sites now surface an explicit,
actionable note naming the real user-global config location and the exact
Axiom servers to add there, closing a gap where a second, independent
gating guard (`workspace-worktree-provision.ts`'s `isNativeMcpTarget`,
discovered during this increment) would otherwise have silently swallowed
the new informative `case`s. `litellm` (and any future undeclared target)
is unchanged. The portable `.axiom/{agents,commands,skills}/` process
surface was confirmed — not assumed — to already be adapter-agnostic and
unconditional, so no native per-IDE skill format work was needed for MCP
reachability in this increment; that remains an explicitly deferred,
separate refinement. Build and every targeted/adjacent test surface (plus
a full `apps/cli` safety sweep) pass with zero regressions.

## General spec integration

Not integrated into `Axiom.Spec/specs/00..08` directly — per this task's
brief, do not edit those files; the orchestrator integrates the batch
later, consistent with how the sibling `INC-20260726-adapter-generators`/
`INC-20260726-adapter-registry-canonical` increments in this same repo
were left for a later integration pass. Stable facts worth carrying into
that future integration pass:

- `NATIVE_MCP_TARGETS` (project-scoped native MCP file gate) is now 7 ids:
  `claude-code`, `cursor`, `copilot-vscode`, `github-copilot`, `opencode`,
  `vscode`, `visual-studio-2026`. `visual-studio-2026`'s `.vs/mcp.json`
  schema is a DOCUMENTED, override-able ASSUMPTION, not independently
  re-verified.
- A second, distinct set, `NATIVE_MCP_INFORMATIVE_TARGETS` (`codex`,
  `antigravity`), covers targets whose real MCP config is user-global —
  Axiom never writes a project file for them, only an informative note
  naming the real location + the servers to add manually. Any future
  caller that dispatches on `NATIVE_MCP_TARGETS` to decide whether to
  invoke `writeNativeMcpConfig` at all should also check
  `NATIVE_MCP_INFORMATIVE_TARGETS`, or the informative note becomes dead
  code for that caller (the exact gap found and fixed in
  `workspace-worktree-provision.ts` during this increment).
- The portable `.axiom/{agents,commands,skills}/` surface
  (`materializeProcessSurfaces`) is confirmed adapter-agnostic and
  unconditional — every repo gets it regardless of which native adapters
  are selected.
- Deferred, explicitly out of scope: native per-IDE skill/agent file
  formats beyond the existing claude-code/opencode/cursor/github-copilot/
  copilot-vscode set; re-verifying Visual Studio 2026's real MCP schema
  independently; updating `packages/adapters/{visual-studio-2026,
  antigravity}`'s own bundled AGENTS.md/AXIOM.md note text (now
  cosmetically stale, not functionally wrong).
