# Increment: Dedicated adapter generators for codex / antigravity / visual-studio-2026

Status: closed
Date: 2026-07-26

## Goal

Give three adapter targets (`codex`, `antigravity`, `visual-studio-2026`)
FIRST-CLASS adapter generator packages — replacing the thin canonical
`AGENTS.md` fallback path (`writeThinCanonicalAgentsMd` +
`renderCanonicalAgentsMd`) — so they materialize a real,
merge-preserving instruction surface exactly like `claude-code`/
`opencode` already do:

- `@axiom/adapters-codex` → writes `.codex/AGENTS.md`
- `@axiom/adapters-antigravity` → writes `.antigravity/AGENTS.md`
- `@axiom/adapters-visual-studio-2026` → writes `.vs/AXIOM.md` (path
  unchanged — doctor and path checks depend on this exact path)

## Context

A precondition increment (INC-1, referenced in this increment's brief
as already done — not separately reconciled into `Axiom.Spec` as of
this writing) had already registered `vscode` and `codex` as
recognized adapter targets in `axiom.yaml#capabilities.adapters` and
wired `vscode` to its real generator
(`@axiom/adapters-vscode#generateVscodeConfig`). `codex`/
`antigravity`/`visual-studio-2026` still fell through to a shared
thin-canonical-AGENTS.md path
(`writeThinCanonicalAgentsMd`/`renderCanonicalAgentsMd` in
`apps/cli/src/commands/workspace-adapters.ts`), which emits a generic,
low-information `AGENTS.md`/`AXIOM.md` with only a project name and an
optional target-specific "extra note" appended after the
`TEAM:CUSTOM` block — not the richer, template-driven,
merge-preserving surface `claude-code`/`opencode` produce.

`@axiom/adapters-claude-code` was used as the reference pattern to
mirror: it is the SINGLE-FILE generator (one `AGENTS.md`, no
skills-lock), simpler than `opencode`'s multi-file
(`AGENTS.md`+`skills-lock.yaml`) pattern.

## Scope

- 3 new packages under `packages/adapters/{codex,antigravity,visual-studio-2026}/`,
  each with `package.json`, `tsconfig.json`, `src/{index,generator,agents-md,types}.ts`,
  and `tests/generator.test.ts` — structurally identical to
  `@axiom/adapters-claude-code` (same `Result<T, AdapterGeneratorError>`
  contract, same 6-state preservation-marker algorithm
  (`classifyMarkers`/`extractTeamBlock`/`renderAxiomBlock`/`mergeBlocks`/
  `renderAgentsMd`/`writeAgentsMd`), same `templateContent`-precedence-
  over-disk-read design). Each package's `renderAxiomBlock` differs
  from claude-code's only in its trailing adapter-specific note (in
  place of claude-code's model-routing "Routing note", which does not
  apply to these targets):
  - codex: `## Codex adapter notes` — identifies that Codex CLI reads
    `AGENTS.md` as its instruction surface, points at the portable
    `.axiom/` surface and the canonical Axiom SDD workflow.
  - antigravity: `## Antigravity adapter notes` — folds in the
    previous `ANTIGRAVITY_EXTRA_NOTE` guidance verbatim-in-substance
    (its MCP config is user-global, `~/.gemini/config/mcp_config.json`,
    `mcpServers`; Axiom never writes there automatically).
  - visual-studio-2026: `## Visual Studio 2026 adapter notes` — folds
    in the previous `VISUAL_STUDIO_2026_EXTRA_NOTE` guidance verbatim-
    in-substance (no verified native MCP schema for this target yet;
    `.vscode/mcp.json` may be readable by it, unverified).
- `apps/cli/src/commands/workspace-adapters.ts`: imports the 3 new
  `generate{Codex,Antigravity,VisualStudio2026}Config` functions;
  replaced the `case 'codex'` / `'antigravity'` / `'visual-studio-2026'`
  dispatch blocks with real generator calls modeled exactly on the
  `case 'claude-code'` block (bundled `templateContent`, `renderContext`,
  error mapping into `warnings`). `writeThinCanonicalAgentsMd` and its
  now-orphaned path/note constants (`ANTIGRAVITY_AGENTS_MD_RELATIVE_PATH`,
  `VISUAL_STUDIO_2026_AXIOM_MD_RELATIVE_PATH`,
  `CODEX_AGENTS_MD_RELATIVE_PATH`, `ANTIGRAVITY_EXTRA_NOTE`,
  `VISUAL_STUDIO_2026_EXTRA_NOTE`, `CODEX_EXTRA_NOTE`) are left in the
  file, unreferenced, with updated doc comments explicitly marking them
  as dead code (kept intentionally, out of this increment's scope to
  remove/refactor — a future adapter target with no dedicated generator
  could still use the helper as a fallback).
- Root `tsconfig.json` + `apps/cli/tsconfig.json`: added `references`
  and `paths` entries for the 3 new packages.
- `vitest.config.ts`: added path aliases for the 3 new packages
  (vitest resolves workspace packages independently of `tsc`'s project
  references).
- `packages/doctor/src/checks.ts` (`TC-009` /
  `runAdapterRuntimeCoverageCheck`): `ADAPTER_PACKAGES` grew from 6 to
  9 (`codex`, `antigravity`, `visual-studio-2026` added); updated
  comments (the "6 packages"/"3 targets sin package" prose was stale).
- `packages/doctor/tests/adapters.test.ts`: updated the `9/9 adapters`
  evidence assertion (was `6/6`).
- `packages/model-routing/src/support-matrix.ts`: comment-only update
  (TC-009 count 6→9); the `SUPPORT_MATRIX`/`SupportLevel` VALUES for
  these 3 targets are UNCHANGED (`'fallback-only'`) — that axis measures
  per-slot model-routing projection, a genuinely separate concern from
  "does this target have a dedicated AGENTS.md generator package",
  which this increment does not touch.
- `apps/cli/tests/workspace-adapters.test.ts` +
  `apps/cli/tests/e2e/adapters.e2e.test.ts`: updated assertions that
  previously expected the thin canonical output (literal
  `${projectName}` string, headings `## Nota sobre MCP (Axiom)` /
  `## Nota sobre soporte de codex (Axiom)`) to expect the real
  generated content (headings `## Antigravity adapter notes` /
  `## Visual Studio 2026 adapter notes` / `## Codex adapter notes`,
  `<!-- AXIOM:GENERATED:START -->` markers). The `projectName` literal
  assertion was removed for antigravity/codex because the shared
  bundled template's placeholders (`{role}`/`{description}`/etc.) are
  only substituted when the caller passes a `renderContext` — these
  particular test calls don't (matching how the pre-existing
  opencode/claude-code assertions in the same file already behave).
- Also updated (documentation-only, no logic change): `docs/README.md`,
  `packages/adapters/README.md`, `packages/installer/src/registry.ts`
  (comments on the `codex`/`antigravity` entries in
  `GENERATED_FILES_BY_TARGET`).

## Non-goals

- Native MCP config projection for `codex`/`antigravity`/
  `visual-studio-2026` (`NATIVE_MCP_TARGETS`) — explicitly out of
  scope per the brief ("native MCP wiring is INC-3, not here").
  `@axiom/model-routing#SUPPORT_MATRIX`'s `'fallback-only'` level for
  these 3 targets is UNCHANGED and still accurate.
- Skills lockfile / multi-file output for these 3 targets — mirrors
  claude-code's single-file pattern deliberately (brief: "mirror the
  SINGLE-FILE generator pattern... simpler than opencode").
- Removing `writeThinCanonicalAgentsMd` or its now-unused constants —
  explicitly deferred by the brief ("if it does [become dead], leave
  it... but note it in the spec").
- Any change to `axiom.yaml#capabilities.adapters`,
  `ADAPTER_TARGETS`, or the adapter registry membership itself — that
  was the precondition increment's scope, already done.

## Acceptance criteria

- [x] `@axiom/adapters-codex` exists, writes `.codex/AGENTS.md`.
- [x] `@axiom/adapters-antigravity` exists, writes `.antigravity/AGENTS.md`.
- [x] `@axiom/adapters-visual-studio-2026` exists, writes `.vs/AXIOM.md`
      (path unchanged).
- [x] Each package mirrors claude-code's file layout, `Result`-based
      error contract, merge-preserving `TEAM:CUSTOM` behavior, and
      `templateContent`-precedence-over-disk-read design.
- [x] `workspace-adapters.ts`'s dispatch for these 3 targets calls the
      real generators (modeled on the `claude-code` case block), no
      longer the thin canonical fallback.
- [x] Root `tsconfig.json`: `references` added for the 3 new packages.
- [x] Workspace package resolution works (`npm install` created the 3
      new `node_modules/@axiom/adapters-*` symlinks via the existing
      `packages/*/*` workspace glob — no `package.json` change needed).
- [x] `packages/doctor`'s TC-009 (`ADAPTER_PACKAGES`) updated 6 → 9;
      its test's evidence assertion updated to `9/9`.
- [x] `apps/cli/tests/e2e/adapters.e2e.test.ts` and
      `apps/cli/tests/workspace-adapters.test.ts` updated to assert the
      real generated content instead of the thin fallback's.
- [x] `npm run build` (`tsc -b`) passes; `dist/index.js` materializes
      for all 3 new packages.
- [x] `npx vitest run packages/adapters packages/doctor
      apps/cli/tests/workspace-adapters.test.ts
      apps/cli/tests/e2e/adapters.e2e.test.ts` — all pass.

## Open questions

None blocking.

## Assumptions

1. **The 3 new packages do NOT include `mcp-json.ts`/`routing-snapshot.ts`**
   (files `@axiom/adapters-claude-code` also has, but which came from
   LATER, unrelated increments — MCP JSON generation and model-routing
   diagnostics). The brief's explicit file list for each new package
   was `package.json`, `tsconfig.json`, `src/{index,generator,agents-md,types}.ts`,
   `tests/generator.test.ts` — matching claude-code's ORIGINAL
   single-file AGENTS.md scope, not its full current file set.
2. **Each package's adapter-specific note replaces claude-code's
   "Routing note"** (model-routing single-mode diagnostic, not
   applicable here) rather than being appended as a wholly separate,
   additional block — keeps each package's `renderAxiomBlock` output
   shape/structure identical to claude-code's, differing only in the
   trailing note's heading + content.
3. **`writeThinCanonicalAgentsMd` and its supporting constants are left
   in place, unreferenced**, per the brief's explicit instruction not
   to remove them — a future adapter target registered without a
   dedicated generator could still use the helper as a fallback.
4. **No `axiom.yaml`/`ADAPTER_TARGETS`/registry membership changes** —
   those were the precondition increment's scope; this increment only
   replaces HOW 3 already-registered targets materialize their output.

## Implementation notes

### Package structure (all 3, mirroring `@axiom/adapters-claude-code`)

- `src/types.ts`: `<Name>Skill`, `<Name>ConfigArgs`,
  `<Name>GeneratorResult`, `AdapterGeneratorError` (kinds
  `template-missing | io-error`), `AgentsMdRenderContext`, plus the
  internal helper types (`RenderAgentsMdArgs`, `WriteAgentsMdArgs`,
  `WriteAgentsMdResult`, `ClassifyMarkersResult`).
- `src/agents-md.ts`: the 6-state preservation-marker machine
  (`classifyMarkers`/`extractTeamBlock`/`renderAxiomBlock`/
  `mergeBlocks`/`renderAgentsMd`/`writeAgentsMd`/
  `substitutePreamblePlaceholders`) — byte-for-byte the same
  algorithm as claude-code's, differing only in `renderAxiomBlock`'s
  trailing adapter note.
- `src/generator.ts`: the top-level `generate<Name>Config` orchestrator
  — `templateContent` precedence over disk-read, `existingFiles`
  in-memory seam, atomic write (tmp + rename), `Result`-wrapped
  errors. Output dir/filename per target:
  `.codex/AGENTS.md` | `.antigravity/AGENTS.md` | `.vs/AXIOM.md`.
- `src/index.ts`: barrel re-exporting the generator + agents-md
  helpers + types.
- `tests/generator.test.ts`: 6 scenarios mirroring claude-code's own
  test file (cold start, `TEAM:CUSTOM` preservation, `AXIOM:GENERATED`
  regeneration, alphabetical capability ordering, `templateContent`
  precedence — 3 sub-cases, `renderContext` placeholder substitution —
  2 sub-cases) plus pure-helper unit tests
  (`readTemplate`/`classifyMarkers`/`renderAxiomBlock`/`mergeBlocks`).

### Workspace resolution

The root `package.json`'s `workspaces` glob is
`["apps/*", "packages/*", "packages/*/*"]` — `packages/adapters/<name>`
already matches `packages/*/*`, so no `package.json` change was
needed; a plain `npm install` from `Axiom/` created the 3 new
`node_modules/@axiom/adapters-{codex,antigravity,visual-studio-2026}`
symlinks. `tsc -b` project references (root `tsconfig.json` +
`apps/cli/tsconfig.json`) and `vitest.config.ts`'s `resolve.alias` were
updated separately (TS project references and vitest aliasing are
independent of npm workspace symlinks and each needed their own
entry).

## Validation

Ran from `C:\repos\Axiom Workspace\Axiom`:

- `npm install` — added 3 packages (workspace symlinks for the 3 new
  adapter packages), no other changes.
- `npm run build` (`tsc -b`) — PASSED, clean, exit 0 (run twice: once
  right after creating the packages, once again after all test-file
  edits — both clean).
- Confirmed `dist/index.js` exists for all 3 new packages:
  `packages/adapters/{codex,antigravity,visual-studio-2026}/dist/index.js`.
- `npx vitest run packages/adapters packages/doctor
  apps/cli/tests/workspace-adapters.test.ts
  apps/cli/tests/e2e/adapters.e2e.test.ts` — **33 files / 331 tests
  passed** (includes the 3 new `packages/adapters/{codex,antigravity,
  visual-studio-2026}/tests/generator.test.ts`, 16 tests each).
- `npx vitest run packages/doctor/tests/adapters.test.ts` (TC-009 guard,
  isolated) — **2/2 tests passed**, evidence confirms `9/9 adapters`.
- Broader safety pass: `npx vitest run apps/cli/tests packages/installer
  packages/install-profiles packages/model-routing` — **145 files /
  1352 tests passed**, zero failures (full `apps/cli` suite, including
  worktree/execution-mode/launcher tests unrelated to this change,
  confirming no regression).

No pre-existing failures were encountered in any touched or adjacent
scope.

## Result

Implemented. `codex`, `antigravity`, and `visual-studio-2026` now have
first-class, dedicated adapter generator packages
(`@axiom/adapters-codex`, `@axiom/adapters-antigravity`,
`@axiom/adapters-visual-studio-2026`), each mirroring
`@axiom/adapters-claude-code`'s single-file, merge-preserving pattern
byte-for-byte except for their trailing adapter-specific note (folding
in — not losing — the guidance the previous thin-canonical fallback
used to emit). `workspace-adapters.ts`'s dispatch for these 3 targets
now calls the real generators, modeled exactly on the pre-existing
`claude-code` case block. `TC-009` (`packages/doctor`) enforces
`src/generator.ts` + `dist/index.js` presence for all 9 dedicated
adapter packages (was 6). `writeThinCanonicalAgentsMd` and its 3
target-specific constant pairs are left in the codebase, unreferenced,
per explicit brief instruction — documented as intentional dead code
in both the source comments and here. Build and the full,
adapter-relevant test surface are green; a broader `apps/cli` +
installer/install-profiles/model-routing safety pass (145 files / 1352
tests) also passed with zero regressions.

## General spec integration

No integration into `Axiom.Spec/general-spec.md` performed. Per this
increment's brief ("Do NOT edit `Axiom.Spec/specs/00..08` — orchestrator
integrates later"), stable knowledge from this increment (and its
precondition, INC-1/`adapter-registry-canonical`, which this increment
builds on but which has not itself been reconciled into
`Axiom.Spec/specs/00..08` yet) is intentionally left for a later batch
integration pass, consistent with how the sibling
`INC-20260724-worktree-mode-selection` increment in this same repo was
integrated "al cierre del batch" rather than per-increment. Stable
facts worth carrying into that future integration pass:
- `codex`/`antigravity`/`visual-studio-2026` each have a dedicated
  `@axiom/adapters-<target>` package (single-file, merge-preserving,
  same contract as `claude-code`).
- `TC-009` (`@axiom/doctor`) now covers 9 adapter packages; the only
  MVP-declared target still without a dedicated package is
  `copilot-vscode`.
- The `SUPPORT_MATRIX`/model-routing `'fallback-only'` level for these
  3 targets is a SEPARATE axis from "has dedicated generator package"
  and remains unchanged by this increment.
