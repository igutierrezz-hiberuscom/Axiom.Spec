# Increment: Launcher adapter routing parity (8 headline adapters + cli)

Status: closed
Date: 2026-07-26

## Goal

Add first-class `AdapterRoutingEntry` entries to `@axiom/launcher`'s
`AXIOM_ADAPTER_ROUTING` so all 8 headline adapter targets (`claude-code`,
`github-copilot`, `vscode`, `opencode`, `cursor`, `antigravity`,
`visual-studio-2026`, `codex` — the same headline set
`INC-20260726-adapter-registry-canonical` wrote to
`axiom.yaml#capabilities.adapters`) plus plain `cli` are selectable in the
`axiom app` launcher and route to the REAL skill id + `workflowCommand` +
`mcpTool` — never the generic `@axiom` clipboard fallback.

## Context

The launcher (`axiom app`)'s adapter selector is populated ENTIRELY from
`AXIOM_ADAPTER_ROUTING.adapters` (`packages/launcher/src/adapter-routing.ts`),
surfaced by `apiGetLauncherData` (`apps/cli/src/commands/app-launcher.ts`) and
rendered dynamically by the front (`apps/cli/static/launcher/launcher.js`).
Before this increment the table had only 3 entries (`claude-code`,
`github-copilot`, `cli`), so selecting `opencode`/`cursor`/`vscode`/
`visual-studio-2026`/`antigravity`/`codex` was impossible — every one of those
ids would hit `resolveActionRouting`'s unknown-adapter branch and degrade to
the generic `@axiom` clipboard fallback. This is a routing-table parity gap,
distinct from (and layered ON TOP OF) the three sibling increments closed the
same day that dealt with the CANONICAL `ADAPTER_TARGETS` registry, workspace
generators, and native MCP config projection:

- `INC-20260726-adapter-registry-canonical` — registered `codex`/`vscode` as
  recognized `ADAPTER_TARGETS` ids and reconciled the headline
  `axiom.yaml#capabilities.adapters` list.
- `INC-20260726-adapter-generators` — gave `codex`/`antigravity`/
  `visual-studio-2026` dedicated `@axiom/adapters-<target>` AGENTS.md/AXIOM.md
  generator packages.
- `INC-20260726-adapter-mcp-parity` — extended native MCP config projection
  (`.vscode/mcp.json`, `.vs/mcp.json`, user-global informative notes) for
  `vscode`/`visual-studio-2026`/`codex`/`antigravity`.

None of those three touched `@axiom/launcher`'s `AXIOM_ADAPTER_ROUTING` — the
table that decides which adapters the WEB LAUNCHER lets a user pick from when
crafting a prompt. This increment closes that remaining gap.

## Scope

- `packages/launcher/src/adapter-routing.ts`: 6 new
  `AdapterRoutingEntry` entries appended to `AXIOM_ADAPTER_ROUTING.adapters`
  (`vscode`, `opencode`, `cursor`, `antigravity`, `visual-studio-2026`,
  `codex`), each with `routingMap: skillRoutingMap()` (reused, unchanged
  function — same real `axiom-sdd-orchestrator`/`axiom-phase-reviewer` skill
  ids + `sdd.transitionApply` mcpTool every existing entry already gets) and
  `agentTuning: { verbosity: 'low', personality: 'pragmatic' }`. The 3
  pre-existing entries (`claude-code`, `github-copilot`, `cli`) are unchanged.
  A routing-model table documenting each new entry's `promptDialect`/
  `launchMechanism`/`defaultAgentMention` choice was added as a doc comment
  directly above the table (see "Routing model" below — mirrored verbatim).
- `apps/cli/src/commands/app-launcher.ts`: added a new `ADAPTER_LABELS`
  (id → friendly display name) map and a `label` field on
  `LauncherDataResponse.adapters[]`, populated via
  `ADAPTER_LABELS[a.id] ?? a.id`. `apiGetLauncherData` already mapped the
  FULL `AXIOM_ADAPTER_ROUTING.adapters` array (no hardcoded subset existed) —
  confirmed, not changed.
- `apps/cli/static/launcher/launcher.js`: the adapter `<select>` already
  rendered dynamically from `S.data.adapters` (no hardcoded 3-item list
  existed) — confirmed, not changed in that respect. Updated the option label
  to prefer the new `a.label` field over the raw `a.id` (falls back to `a.id`
  if `label` is absent, e.g. against an older server response shape).
- `packages/launcher/tests/adapter-routing.test.ts`: extended to cover all 9
  adapters — a new exact-membership assertion (`adapters.map(id) === [8
  headline ids, 'cli']`), the skill/phase-reviewer routing assertions
  parameterized over all 8 skill-routed adapters (was 2), and a new
  "no headline adapter or cli ever hits the unknown-adapter fallback" test
  that resolves EVERY `ACTION_RECONCILIATION` key against every one of the 9
  adapters and asserts a defined `target`/`mcpTool` with both fallback flags
  `false`.

## Non-goals

- Touching the canonical `ADAPTER_TARGETS` registry, workspace-adapter
  generators, or native MCP config projection — those are the 3 sibling
  increments' scope, already closed, and are a DIFFERENT registry from
  `AXIOM_ADAPTER_ROUTING` (that one gates `axiom init --target`/workspace
  materialization; this one gates the web launcher's prompt-crafting adapter
  picker).
- Changing `promptDialect`/`launchMechanism`/`defaultAgentMention` semantics
  or `renderMention`'s per-dialect rendering logic — reused as-is.
- Adding per-adapter routing maps that differ from `skillRoutingMap()` for
  any of the 6 new entries — every headline adapter routes identically
  (real skill ids + mcpTool); only the header/mention cosmetics differ.
- Extending `packages/launcher/tests/adapter-routing-snapshot.test.ts` (the
  2-adapter body-invariant snapshot test) — it does not hardcode an adapter
  count and continues to pass unchanged; extending it to a 3rd/4th adapter
  was judged out of proportion (the invariant it proves — same body across
  adapters — is already covered structurally by `skillRoutingMap()` reuse
  and by the new fallback-membership test).
- Any change to `apps/cli/static/launcher/index.html` — it has no hardcoded
  adapter list to begin with (confirmed).

## Acceptance criteria

- [x] `AXIOM_ADAPTER_ROUTING.adapters` contains exactly 9 entries, in order:
      `claude-code, github-copilot, vscode, opencode, cursor, antigravity,
      visual-studio-2026, codex, cli`.
- [x] Every one of the 8 headline adapters routes every
      `ACTION_RECONCILIATION` action to a real skill id
      (`axiom-sdd-orchestrator` or, for `review-*`, `axiom-phase-reviewer`)
      + the `sdd.transitionApply` mcpTool — never the unknown-adapter
      fallback.
- [x] `cli` is unchanged and still routes to real `axiom-<kind> <sub>`
      invocations.
- [x] `apiGetLauncherData` surfaces all 9 adapters with a human-readable
      `label` (new field), with no hardcoded adapter-id list anywhere in
      `app-launcher.ts` outside the single `ADAPTER_LABELS` map.
- [x] The launcher front (`launcher.js`) renders the adapter selector
      dynamically from the server response and prefers `label` over `id`;
      default selection stays `claude-code`.
- [x] `packages/launcher/tests/adapter-routing.test.ts` covers all 9
      adapters (membership, skill routing, review routing, no-fallback).
- [x] `npm run build` (`tsc -b`) passes clean.
- [x] Targeted + broader test sweeps pass (see Validation).

## Open questions

None blocking.

## Assumptions

1. **`opencode`'s `defaultAgentMention` is `@axiom-sdd-orchestrator`** (the
   materialized agent mention), per the locked design brief — OpenCode
   invokes its own materialized agent by mention rather than a generic
   `@axiom` participant. This is a cosmetic header hint only; the
   `routingMap` (the acceptance bar) is `skillRoutingMap()`, identical to
   every other headline adapter.
2. **`cursor` uses `promptDialect: 'slash-command'` /
   `launchMechanism: 'skill-invocation'`** (like `claude-code`), per the
   locked design brief — Cursor supports slash-command + rules-style
   invocation, distinguishing it from the chat-mention adapters
   (`vscode`/`antigravity`/`visual-studio-2026`/`codex`/`github-copilot`).
3. **`vscode`/`antigravity`/`visual-studio-2026`/`codex` all get the generic
   `@axiom` `defaultAgentMention`** with `chat-mention`/`chat-participant` —
   per the locked design brief, since none of these 4 has a more specific
   mention convention documented, and the brief explicitly said "if any of
   these labels reads oddly, keep it — the routingMap correctness is the
   acceptance bar, not the mention string."
4. **`ADAPTER_LABELS` was added** (not skipped) — the brief listed exact
   friendly names for every adapter id ("Claude Code", "GitHub Copilot", "VS
   Code", "OpenCode", "Cursor", "Antigravity", "Visual Studio 2026", "Codex",
   "CLI"), a strong signal the raw kebab-case ids (e.g.
   `visual-studio-2026`) were judged insufficiently readable in the
   selector. The map lives solely in `app-launcher.ts` (server side); the
   front only consumes the resulting `label` field, so no adapter-id list is
   duplicated in `launcher.js`.
5. **`packages/launcher/tests/adapter-routing-snapshot.test.ts` was left
   unchanged** — see Non-goals. It asserts a structural invariant (2 known
   adapters differ ONLY in header/launch-verb) that reused-map data doesn't
   put at risk; the new fallback-membership test in
   `adapter-routing.test.ts` is the more precise proof for the 6 new
   entries specifically.

## Routing model

Every headline adapter routes through the SAME reused `skillRoutingMap()` (a
`routingMap` = `Record<action-key, RoutingTarget>` where every target is
`{ kind: 'skill', id: 'axiom-sdd-orchestrator' | 'axiom-phase-reviewer',
workflowCommand?, mcpTool: 'sdd.transitionApply' }`) — the routingMap is the
correctness bar; `promptDialect`/`launchMechanism`/`defaultAgentMention` are
cosmetic header hints for each surface's own convention:

| id                   | promptDialect   | launchMechanism    | defaultAgentMention        | routingMap        |
|----------------------|-----------------|--------------------|-----------------------------|-------------------|
| `claude-code`        | slash-command   | skill-invocation   | `/axiom-sdd-orchestrator`   | `skillRoutingMap()` (unchanged) |
| `github-copilot`     | chat-mention    | chat-participant   | `@axiom`                    | `skillRoutingMap()` (unchanged) |
| `vscode`             | chat-mention    | chat-participant   | `@axiom` (Copilot Chat)     | `skillRoutingMap()` (new) |
| `opencode`           | chat-mention    | chat-participant   | `@axiom-sdd-orchestrator` (materialized agent mention) | `skillRoutingMap()` (new) |
| `cursor`             | slash-command   | skill-invocation   | `/axiom-sdd-orchestrator` (slash + rules) | `skillRoutingMap()` (new) |
| `antigravity`        | chat-mention    | chat-participant   | `@axiom` (Gemini/Antigravity chat) | `skillRoutingMap()` (new) |
| `visual-studio-2026` | chat-mention    | chat-participant   | `@axiom` (VS Copilot chat)  | `skillRoutingMap()` (new) |
| `codex`              | chat-mention    | chat-participant   | `@axiom` (Codex CLI consumes the crafted prompt directly) | `skillRoutingMap()` (new) |
| `cli`                | shell           | cli-exec           | `$ axiom`                   | `cliRoutingMap()` (unchanged) |

## Implementation notes

- `skillRoutingMap()`/`cliRoutingMap()` are private (non-exported) helper
  functions local to `adapter-routing.ts`; no export was needed since the 6
  new entries live in the same file/module as the table they extend.
- `ADAPTER_LABELS` (`apps/cli/src/commands/app-launcher.ts`): a single
  `Readonly<Record<string, string>>` consumed only by `apiGetLauncherData`;
  an id absent from the map falls back to the raw id (defensive, future-proof
  for a not-yet-labelled adapter).
- `launcher.js`'s adapter `<select>` option text became
  `label + ' (' + promptDialect + ')' + tuningLabel` (was `id + ...`);
  `label` defaults to `a.id` client-side too, so an older/mismatched server
  response never breaks rendering.

## Validation

Ran from `C:\repos\Axiom Workspace\Axiom`:

- `npm run build` (`tsc -b`) — PASSED, clean, exit 0.
- `npx vitest run packages/launcher/tests/adapter-routing.test.ts
  apps/cli/tests/app-launcher.test.ts` (the exact files the grep for
  `adapter-routing|getLauncherData|AXIOM_ADAPTER_ROUTING|launcher-data`
  turned up) — **2 files / 23 tests passed** (10 in
  `adapter-routing.test.ts`, up from 5, covering the new membership/
  no-fallback assertions across all 9 adapters; 13 in `app-launcher.test.ts`,
  unchanged, confirming the added `label` field does not break the existing
  `toContain('claude-code')`-style assertion).
- `npx vitest run packages/launcher` (full package sweep, extra safety
  pass) — **6 files / 44 tests passed**, including the untouched
  `adapter-routing-snapshot.test.ts` (7/7, confirmed unaffected).
- `npx vitest run apps/cli/tests` (full suite, extra safety pass) — **125
  files / 1167 tests passed**, zero failures — includes
  `launcher-front-no-vscode.test.ts` (unrelated to the `vscode` adapter id;
  it asserts zero VSCode-webview-API coupling in the shipped front, still
  green) and every launcher/execution-mode/worktree test.

No pre-existing failures were encountered in any touched or adjacent scope.

## Result

Implemented. `AXIOM_ADAPTER_ROUTING.adapters` now lists 9 entries — the 8
headline adapter targets (`claude-code`, `github-copilot`, `vscode`,
`opencode`, `cursor`, `antigravity`, `visual-studio-2026`, `codex`) plus
`cli` — all reusing the SAME `skillRoutingMap()`/`cliRoutingMap()` data, so
every action for every adapter resolves to the real
`axiom-sdd-orchestrator`/`axiom-phase-reviewer` skill id plus the
`sdd.transitionApply` mcpTool, never the generic clipboard fallback. The
launcher's `apiGetLauncherData` surfaces all 9 with a new human-readable
`label` field (`ADAPTER_LABELS`, e.g. "Visual Studio 2026", "OpenCode"); the
front already rendered the adapter selector dynamically from the server
response and now prefers `label` over the raw id, with `claude-code` staying
the default selection. Tests were extended to prove all 9 adapters, not just
2, never hit the unknown-adapter fallback. Build and every targeted/adjacent
test surface pass with zero regressions.

## General spec integration

Not integrated into `Axiom.Spec/specs/00..08` directly — the 3 sibling
`INC-20260726-adapter-*` increments in this same batch were also left
unintegrated pending a later batch pass (see their own "General spec
integration" sections), and this increment layers directly on top of that
same headline-adapter vocabulary. Stable facts worth carrying into that
future integration pass:

- `@axiom/launcher`'s `AXIOM_ADAPTER_ROUTING` (the web launcher's adapter
  picker) is now aligned with the headline 8-adapter set
  (`axiom.yaml#capabilities.adapters`) plus `cli` — previously it only
  covered 3 of those ids. Any future adapter added to the headline set
  should also get an `AdapterRoutingEntry` here (reusing `skillRoutingMap()`
  unless it needs a genuinely different routing target), or it silently
  degrades to the clipboard fallback in the launcher UI.
- The launcher's `LauncherDataResponse.adapters[]` now carries a `label`
  field (human-readable display name), sourced from a single
  `ADAPTER_LABELS` map in `app-launcher.ts` — the front never hardcodes
  adapter display names.
