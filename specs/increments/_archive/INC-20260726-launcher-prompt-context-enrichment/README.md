# Increment: Launcher prompt context enrichment (WHERE / MCP+tool / skill)

Status: closed
Date: 2026-07-26

## Goal

Make every prompt the web launcher (`axiom app`) crafts tell the receiving
agent, adapter-neutrally: WHERE to read (the artifact's real spec folder/
README/metadata location, reusing the existing folder-per-artifact
convention), WHICH MCP servers + confirmed-mutation tool to use, and WHICH
skill to apply — closing the gap where `buildContext`
(`apps/cli/src/commands/app-launcher.ts`) populated ONLY `normalizedId`/
`title` on `ResolvedLauncherContext`, even though the routing layer
(`adapter-routing.ts`) already resolves the skill id/mcpTool per action.

## Context

`craftPrompt` (`packages/launcher/src/adapter-routing.ts`) already resolves,
per action + adapter, a `RoutingTarget` carrying `kind` (`'skill' | 'command' |
'mcp-tool'`), the real skill id (`axiom-sdd-orchestrator` /
`axiom-phase-reviewer`) or CLI command, and the confirmed-mutation MCP tool
(`sdd.transitionApply`) — but none of that ever reached the crafted prompt
TEXT; it only lived in the `CraftedPrompt.target` metadata returned alongside
the prompt. Separately, `ResolvedLauncherContext` (`packages/launcher/src/
types.ts`) already declared `specReadmePath`/`specFolderPath`/`metadataPath`
(and `planReadmePath`/`rolePlanPath`/etc.) — and `prompt-builder.ts`'s
context-line builders already rendered them when present — but
`buildContext` (`apps/cli/src/commands/app-launcher.ts`, the ONLY place that
constructs a `ResolvedLauncherContext`) never populated any of them, so those
lines never appeared in a real crafted prompt. Two independent,
already-wired-but-never-fed capabilities; this increment feeds both.

## Scope

- `packages/launcher/src/types.ts`: new `PromptToolingHints` type
  (`mcpServers?`, `mcpTool?`, `skillId?`, all optional) + a new optional
  `tooling?: PromptToolingHints` field on `PromptBuildOptions`. Fully
  backward-compatible — omitted `tooling` renders no block at all.
- `packages/launcher/src/prompt-builder.ts`: new `buildToolingBlock` renders
  a single "Herramientas y ubicacion:" block (MCP servers, mcpTool, skillId
  when present, plus a pointer-only "Lecturas" line — never re-prints a
  literal spec/metadata path already shown by the context-line/notes blocks).
  Wired into both the structured-body path (placed right after the
  context-line block) and the `review` branch (after `buildReviewPrompt`'s
  body).
- `packages/launcher/src/adapter-routing.ts`: new exported
  `AXIOM_MANAGED_MCP_SERVERS = ['sdd-mcp-server', 'spec-mcp-broker']`
  constant (override-able via a new `CraftPromptOptions.mcpServers`); new
  private `buildToolingHints(target, mcpServers)` derives `mcpTool`/`skillId`
  from the already-resolved `RoutingTarget` (`mcpTool = target?.mcpTool`;
  `skillId` set only when `target?.kind === 'skill'`); `craftPrompt` always
  builds and passes `tooling` into `buildPrompt`.
- `apps/cli/src/commands/app-launcher.ts`: `buildContext` now takes
  `(projectRoot, action, values)` (was `(values)`); new private
  `resolveArtifactKindForContext` (family → `ArtifactKind`, with an id-prefix
  fallback for `back`/`front`/`e2e`/`review` families, mirroring
  `axiom-role.ts`'s `inferArtifactKind` heuristic) and `toRepoRelativePath`
  helpers. `buildContext` now sets `specFolderPath`/`specReadmePath`/
  `metadataPath` via `resolveArtifactDir` (`@axiom/workflow`) +
  `resolveSpecArtifactRelPath` — the SAME resolution the registry endpoint
  (`listKind`) already uses — or, when `id` is empty, pushes an instructional
  note (see below) instead of a fake path. The single call site
  (`apiCraftLauncherPrompt`) was updated to pass `resolved.projectRoot` and
  `action`.
- Tests: `packages/launcher/tests/prompt-builder.test.ts` (5 new cases),
  `packages/launcher/tests/adapter-routing-snapshot.test.ts` (1 new case +
  2 existing cases updated for the new block), `apps/cli/tests/
  app-launcher.test.ts` (5 new cases). Snapshots regenerated.

## Non-goals

- Resolving `planReadmePath`/`planMetadataPath`/`rolePlanPath`/
  `repoHintPath`/`contextFolderPath`/`supportingContextPaths`/`originId` — out
  of this increment's "at minimum the spec folder/README/metadata" bar; a
  future increment can extend `buildContext` to also resolve plan/role-plan
  locations (reading `metadata.yml`'s `links.planId` etc.) the same way.
  `rolePlanPath`/`planReadmePath` stay `undefined` for now — this is a strict
  narrowing of scope, not a regression (they were already always `undefined`
  before this increment).
- Changing `buildSpecificationNotes`/`buildPlanNotes`/`buildImplementationNotes`'s
  existing note-filter predicate (`note.startsWith('No se ha podido resolver
  automaticamente el ID')`) — the new "id not yet assigned" note reuses that
  EXACT existing prefix (it is semantically accurate: without an id there is
  no way to auto-resolve the final folder) rather than widening the filter.
- Surfacing `resolvedContext.notes` in `review`-family prompts —
  `buildReviewPrompt` has never read `resolvedContext.notes` (a pre-existing,
  independent gap); untouched here.
- An `artifactExists`-driven "folder not created yet" note for the
  id-present-but-not-yet-created case — the resolved path is already correct
  and visible via `specFolderPath`/`specReadmePath`/`metadataPath`
  regardless of on-disk existence, and any additional note text would be
  silently dropped by the same notes filter above for spec/plan/impl
  prompts, so it was cut rather than shipped as a functionally-dead line.
- Any change to `LAUNCHER_ACTION_COMMAND`/`buildCliArgs`/`executeSubcommand` —
  execution is untouched; only the READ-ONLY `craft`/prompt-authoring path is
  affected.

## Acceptance criteria

- [x] Every prompt crafted via `craftPrompt` (any action, any adapter) that
      resolves an artifact id gets `specFolderPath`/`specReadmePath`/
      `metadataPath` populated with the real, repo-relative,
      convention-derived location (verified against the SAME
      `resolveArtifactDir`/`resolveSpecArtifactRelPath` the registry endpoint
      uses — no hand-rolled path scheme).
- [x] When the id is empty (artifact not yet created), the crafted prompt
      contains a clear instruction naming the kind-level folder instead of a
      fake per-id path.
- [x] Every crafted prompt contains one adapter-neutral "Herramientas y
      ubicacion:" block listing the managed MCP servers, the resolved
      `mcpTool`, and the `skillId` (when the target is skill-routed).
- [x] The block is IDENTICAL (byte-for-byte) across every skill-routed
      adapter (`claude-code`, `github-copilot`, `opencode` proven directly;
      `vscode`/`cursor`/`antigravity`/`visual-studio-2026`/`codex` share the
      exact same `skillRoutingMap()` data so are structurally identical too).
- [x] The `cli` adapter's block omits the skill line (no `skillId` for a
      `command`-kind target) while still listing the MCP servers/mcpTool —
      documented as the one intentional, expected adapter-specific carve-out.
- [x] No duplication: a literal spec/metadata path string appears at most
      once per prompt (in the context-line block); the tooling block only
      points at it.
- [x] `npm run build` (`tsc -b`) passes clean.
- [x] Targeted + broader test sweeps pass (see Validation).

## Open questions

None blocking.

## Assumptions

1. **Existing `ResolvedLauncherContext` fields were reused, not duplicated.**
   `specReadmePath`/`specFolderPath`/`metadataPath` already existed on the
   type (and were already consumed by every context-line builder in
   `prompt-builder.ts`) — `buildContext` simply never populated them. No new
   fields were added for Part A; the brief's suggested field names
   (`specPath?`, `metadataPath?`) were superseded by the pre-existing,
   already-wired ones.
2. **The "id not yet assigned" note reuses the existing note-filter prefix.**
   `buildSpecificationNotes`/`buildPlanNotes`/`buildImplementationNotes` only
   surface `resolvedContext.notes` entries starting with `'No se ha podido
   resolver automaticamente el ID'` — any other note text is silently
   dropped for spec/bug/plan/impl prompts. Rather than widening that filter
   (a broader, riskier change touching 3 existing, tested predicates), the
   new note's wording starts with that exact, semantically-accurate prefix.
3. **Artifact-kind inference for `back`/`front`/`e2e`/`review` actions
   mirrors `axiom-role.ts`'s existing `inferArtifactKind` heuristic**
   (`bug*` → `'bug'`, `plan*` → `'plan'`, else `'increment'`) — duplicated as
   an independent 3-line helper in `app-launcher.ts` rather than exporting/
   importing across the two unrelated command modules for a heuristic this
   small.
4. **`AXIOM_MANAGED_MCP_SERVERS = ['sdd-mcp-server', 'spec-mcp-broker']`** —
   Axiom's two documented managed MCP servers (see `axiom.config/
   mcp-manifest.yaml`, `packages/mcp-server`). Modeled as an override-able
   `CraftPromptOptions.mcpServers`, not a hidden constant, so a project
   shipping its own MCP topology can override the list without forking
   `craftPrompt`.
5. **Plan/role-plan location resolution is deferred** — see Non-goals;
   `planReadmePath`/`rolePlanPath`/etc. stay unresolved in this increment,
   consistent with the "at minimum" scope in the brief.

## Implementation notes

### The tooling block

Rendered by `buildToolingBlock` (`prompt-builder.ts`), placed immediately
after the context-line block in the structured body, and after the review
body in the `review` branch:

```
Herramientas y ubicacion:
- MCP disponibles: sdd-mcp-server, spec-mcp-broker
- Herramienta MCP para mutaciones confirmadas: sdd.transitionApply
- Skill a aplicar: axiom-sdd-orchestrator
- Lecturas: usa las rutas de spec/plan/metadata ya listadas en este prompt; no las recalcules.
```

For the `cli` adapter (`command`-kind target), the "Skill a aplicar" line is
omitted; everything else is identical. When no read location was resolved at
all (e.g. the id-empty case), the last line instead reads "aun no hay rutas
resueltas en este prompt; revisa las notas del launcher antes de asumir
ubicaciones."

### Location resolution (`buildContext`)

- `id` present: `specFolderPath` = `path.relative(projectRoot,
  resolveArtifactDir(projectRoot, kind, id, resolveSpecArtifactRelPath(projectRoot)))`
  (forward-slash normalized); `specReadmePath`/`metadataPath` append
  `/README.md`/`/metadata.yml`. Set unconditionally — whether or not the
  folder already exists on disk (it is still the correct, action-created
  location).
- `id` empty: same `resolveArtifactDir` call with the literal string `'<id>'`
  as the id segment (reusing the exact same convention, never a parallel
  scheme), turned into an instructional note instead of a path field.

### Cross-adapter invariance proof

`adapter-routing-snapshot.test.ts`'s existing "the CLI adapter crafts the
same body..." test was updated (not just re-snapshotted): it now strips the
"Skill a aplicar" line before comparing `cli` vs `claude-code` bodies, and
separately asserts that line's presence/absence explicitly — documenting the
ONE expected adapter-specific difference instead of silently breaking the
prior byte-equality assertion.

## Validation

Ran from `C:\repos\Axiom Workspace\Axiom`:

- `npm run build` (`tsc -b`) — PASSED, clean, exit 0 (run twice: after Part A
  and after Part B).
- `npx vitest run packages/launcher apps/cli/tests/app-launcher.test.ts` —
  **7 files / 68 tests passed** (`prompt-builder.test.ts` 4→9,
  `adapter-routing-snapshot.test.ts` 7→8 with 2 pre-existing cases updated,
  `app-launcher.test.ts` 13→18). Snapshots regenerated
  (`adapter-routing-snapshot.test.ts.snap`: 2 updated, 1 obsolete entry
  removed for the renamed CLI-body-invariance test).
- `npx vitest run apps/cli/tests` (full suite, safety pass) — **125 files /
  1172 tests passed**, zero failures, zero regressions.

No pre-existing failures were encountered in any touched or adjacent scope.

## Result

Implemented. Every prompt the launcher crafts now carries a real,
repo-relative spec location (or a clear "not yet assigned" instruction) plus
an adapter-neutral "Herramientas y ubicacion" block naming the managed MCP
servers, the confirmed-mutation `mcpTool`, and the skill to apply — with the
`cli` adapter's single, documented, intentional carve-out (no skill line).
Build and the full `apps/cli` + `packages/launcher` suites pass with zero
regressions.

## General spec integration

Not integrated into `Axiom.Spec/specs/00..08` directly (out of this
increment's instructed scope — those files were not touched). Stable facts
worth carrying into a future integration pass, alongside the sibling
`INC-20260726-launcher-adapter-routing-parity`/`INC-20260726-adapter-*`
batch already flagged as pending:

- Every crafted launcher prompt now embeds a "Herramientas y ubicacion"
  block (MCP servers + `mcpTool` + `skillId`) derived from the SAME
  `RoutingTarget` the routing table already resolved — no new routing
  concept, just newly-surfaced existing data.
- `ResolvedLauncherContext.specFolderPath`/`specReadmePath`/`metadataPath`
  are now genuinely populated by `buildContext`, reusing
  `@axiom/workflow`'s `resolveArtifactDir` + `resolveSpecArtifactRelPath` —
  any future context-enrichment work (plan/role-plan paths) should follow
  the same reuse pattern rather than hand-rolling new path logic.
- `AXIOM_MANAGED_MCP_SERVERS` (`sdd-mcp-server`, `spec-mcp-broker`) is now a
  named, override-able constant in `@axiom/launcher` — the canonical pair a
  future MCP-topology doc pass should reference.
