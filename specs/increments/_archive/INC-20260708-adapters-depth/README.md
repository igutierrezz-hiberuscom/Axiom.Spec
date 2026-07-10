# Increment: Adapter depth (claude-code routing snapshot, vscode/VS2026/antigravity depth, adapter e2e)

Status: closed
Date: 2026-07-08

## Goal

Deepen the uneven adapter surface (`packages/adapters/*`) so that
claude-code, vscode (`copilot-vscode`), visual-studio-2026, and
antigravity each go beyond their current bare-minimum output, WITHOUT
fabricating native capability that does not exist — and add a hermetic
adapter e2e test that proves the generated file set is correct and
consistent per adapter target.

## Context

Read first: `packages/adapters/*`, `packages/model-routing/src/{slots,
opencode-projection,effective,resolver,fallback,support-matrix,
checks}.ts`, `packages/installer/src/registry.ts`
(`GENERATED_FILES_BY_TARGET`), `apps/cli/src/commands/
native-mcp-config.ts`, `apps/cli/src/commands/workspace-adapters.ts`,
`apps/cli/src/commands/workspace-mcp.ts`, `apps/cli/src/commands/
model.ts`. Prior increment `INC-20260708-mcp-native-config-mapping`
(archived) already mapped Axiom's two MCP entries onto each tool's
verified native MCP schema (claude-code/cursor `mcpServers`,
copilot-vscode/github-copilot `.vscode/mcp.json` `servers`+
`type:'stdio'`, opencode `opencode.json`), written into every repo of
the workspace.

### Critical finding that reshapes this increment's scope

The parent brief asked for "per-slot model routing projection for
claude-code (mirroring opencode-projection)". Investigation found this
directly **contradicts a deliberately locked architectural decision**
spanning five files, multiple prior closed increments (0026/A5,
0031/E), and dedicated tests:

- `packages/model-routing/src/support-matrix.ts`: `SUPPORT_MATRIX`
  hard-codes `'claude-code': 'single-mode'` with an explicit comment:
  *"claude-code: routing global, sin per-slot; adapter materializa un
  único AGENTS.md con línea `## Routing note` que documenta el
  contrato"*.
- `packages/model-routing/src/fallback.ts` (`applyFallback`): for
  `supportLevel === 'single-mode'`, ALWAYS returns `{ modelClass:
  'medium', fellBack: true, fallbackReason:
  'per-slot-routing-unsupported' }` — this is a total function, not a
  gap to fill in.
- `packages/model-routing/src/resolver.ts` (`resolveSlot`): branches on
  `applyFallback` BEFORE ever consulting the policy; `single-mode`
  targets never reach the per-slot policy lookup, by design.
- `packages/model-routing/src/checks.ts` (`checkRoutingConsistent`,
  `MRC-004`): explicitly asserts `single-mode`/`fallback-only` targets
  get a `warn` ("el routing por slot cae a 'medium' por fallback...
  para routing granular, usá un target con support multi-mode (e.g.
  opencode)") — this is a tested, intentional outcome, not an
  oversight.
- `packages/adapters/claude-code/src/agents-md.ts`
  (`renderAxiomBlock`): emits a literal `## Routing note` /
  "Single-mode: routing global del target. Sin per-slot." block,
  described in a code comment as the definitive contract for
  claude-code.

Additionally, no verified native schema for per-agent/per-task model
routing exists in the Claude Code CLI (unlike opencode's
`opencode.json`, which genuinely supports per-agent `model` fields).
Claude Code's real native model configuration is a single
project/session-level setting, not a 7-slot routing table.

**Decision (resolved without stopping to ask):** do NOT invent
per-slot routing capability for claude-code. Instead, add a
**diagnostic routing-snapshot projection** for claude-code — a
`.claude/model-routing.json` file, structurally mirroring
`OpencodeProjectionPayload`, that transparently records what
`single-mode` resolution actually produces (all 7 slots -> `medium`,
`fellBack: true`, `fallbackReason: 'per-slot-routing-unsupported'`).
This gives an operator/tool visibility into the routing decision
without claiming claude-code can act on a per-slot basis — the
opposite of the brief's literal ask, but the only choice consistent
with "do NOT invent beyond what's verified" and with five locked
decisions already tested elsewhere in this codebase. This is
documented here as an explicit, honest deviation from the brief.

### Verified native config facts reused from
`INC-20260708-mcp-native-config-mapping` (unchanged, not re-verified
here)

- claude-code: `.claude/AGENTS.md` (adapter output) + `.mcp.json`
  (`mcpServers`, native MCP — already wired).
- vscode / copilot-vscode: `.vscode/copilot-instructions.md` +
  `.vscode/settings.json` (adapter output, already existed) +
  `.vscode/mcp.json` (`servers`+`type:'stdio'`, native MCP — already
  wired, SHARED with `github-copilot`'s native MCP file but NOT with
  its `.github/copilot-instructions.md` adapter output, which is a
  separate target/file).
- visual-studio-2026: native MCP schema for `visual-studio-2026` was
  explicitly left UNVERIFIED by the prior increment (no file emitted,
  warning only). No new verification was performed in this increment
  either (see "Honest gaps" below) — `.vs/AXIOM.md` (adapter output)
  is deepened; `.vs/mcp.json` native MCP is NOT added because it
  remains unverified, and inventing it would repeat the exact mistake
  this increment exists to avoid.
- antigravity: MCP config is documented elsewhere as user-global
  (`~/.gemini/config/mcp_config.json`), out of scope for a per-project
  write; this increment does not touch MCP for antigravity at all
  (only deepens `.antigravity/AGENTS.md`).

## Scope

1. **`packages/model-routing/src/claude-code-projection.ts`** (new):
   `projectToClaudeCode({ rootPath, projectName, store, target })`,
   structurally mirroring `projectToOpencode` (same `Result` shape,
   same no-op-if-wrong-target behavior, same error kind
   `'projection-failed'`), writing `<root>/.claude/model-routing.json`
   with the SAME `slots` shape as `OpencodeProjectionPayload` (project,
   target, supportLevel, slots, projectedAt). Because `single-mode`'s
   `resolveAllSlots` always yields `medModelClass: 'medium'`,
   `fellBack: true` for every slot, the payload is intentionally
   uniform — that uniformity IS the honest signal (a consumer reading
   this file sees immediately that per-slot routing is not active for
   this target, rather than the file being silently absent).
   Re-exported from `@axiom/model-routing`'s barrel (`index.ts`).
2. **Wire into `axiom model validate`** (`apps/cli/src/commands/
   model.ts`, `runModelValidate`): after the existing `if (target ===
   'opencode')` projection block, add an analogous `if (target ===
   'claude-code')` block calling `projectToClaudeCode` and reporting
   the same `✓ claude-code projection: <path> (...)` / `⚠ ... falló`
   line shape. Optional, best-effort (warn not fail), same pattern as
   opencode.
3. **`packages/adapters/claude-code/src/routing-snapshot.ts`** (new):
   a `loadRoutingSnapshot`/`getSnapshotModelClass` pair mirroring
   `@axiom/adapters-opencode/routing-snapshot.ts` exactly (same
   `RoutingSnapshotPayload` shape, same absent/malformed/ok
   discriminated result), reading from `.claude/model-routing.json`.
   Exported from the package's barrel. Not wired into
   `generateClaudeCodeConfig`'s output content in this increment (the
   `## Routing note` block already documents single-mode accurately;
   wiring the snapshot into the rendered AGENTS.md is future work, not
   required by the acceptance criteria below — kept out to avoid
   touching the byte-exact preservation-marker renderer under time
   pressure).
4. **vscode/copilot-vscode depth**: no code change needed —
   `.vscode/settings.json` + `.vscode/copilot-instructions.md` (adapter
   output) and `.vscode/mcp.json` (native MCP) already exist and are
   already wired (verified by reading `copilot-vscode/src/generator.ts`
   equivalent path via `writeCopilotInstructions` +
   `native-mcp-config.ts`). This increment adds the e2e coverage (see
   §3 below) that was missing, rather than duplicate generator work.
5. **visual-studio-2026 / antigravity depth**: both currently only get
   `writeThinCanonicalAgentsMd` (`workspace-adapters.ts`) — a single
   generic canonical AGENTS.md/AXIOM.md with no target-specific
   content. Deepened in place (same function, richer per-target
   preamble) rather than adding new dedicated generator packages
   (no new npm deps, no speculative architecture per AGENTS.md
   guardrails): `writeThinCanonicalAgentsMd` now accepts an optional
   `extraNote` string appended after the canonical block, used to:
   - `antigravity`: note that MCP servers for Axiom are NOT
     auto-configured (global-config isolation) and point at
     `~/.gemini/config/mcp_config.json` as the manual/opt-in merge
     target — no code writes to that path (isolation preserved, matches
     the brief's "default = do NOT touch the global file").
   - `visual-studio-2026`: note that native MCP is unverified for this
     target today and that `.vscode/mcp.json` (if `copilot-vscode`/
     `github-copilot` is ALSO selected) is the closest verified
     fallback VS2026 can read, per Microsoft's documented VS2026 MCP
     config search paths — cited as informational, not implemented as
     an auto-write (would require verifying VS2026 actually reads
     `.vscode/mcp.json` in this exact repo layout, which was not
     independently re-verified in this increment; kept as a text note,
     not a behavior change).
6. **Adapter e2e**: new `apps/cli/tests/e2e/adapters.e2e.test.ts`.
   Scaffolds one workspace (single repo, control-only, matching the
   existing e2e's tmp-dir pattern) with ALL 8 `ADAPTER_TARGETS`
   selected, runs `generateWorkspaceAdapters` + `runModelValidate`
   (for the `opencode` and `claude-code` sub-cases) directly
   in-process (hermetic — no real IDE launch), and asserts per adapter:
   - Every path listed in `GENERATED_FILES_BY_TARGET[target]` exists
     and is non-empty.
   - `AGENTS.md`/`AXIOM.md`-shaped files, FOR TARGETS THAT USE THE
     CONVENTION (opencode, claude-code, antigravity,
     visual-studio-2026), contain the canonical
     `AXIOM:GENERATED:START/END` markers. `cursor`'s `AGENTS.md` is
     DELIBERATELY excluded from this check — it uses a simpler,
     marker-less MVP format by its own package's design (README: "Sin
     preservation-marker"), and asserting the canonical marker on it
     would itself have been an invented, unverified uniformity claim.
   - The claude-code and opencode `model-routing.json` snapshots (this
     increment's new artifacts), once `axiom model validate` is run,
     contain all 7 `SLOT_TAXONOMY` keys.
   - `litellm` produces zero files (AD-02, unchanged).
   Does not attempt to spawn/launch any IDE; reuses the same
   `isNonEmptyFile`/`readJson`/`readYaml` helpers pattern already used
   by `workspace-mcp.e2e.test.ts` (duplicated locally rather than
   extracted to a shared test-util, to avoid touching the existing e2e
   file per the "small, focused changes" guardrail).

### Two genuine pre-existing bugs found and fixed while building the e2e

Running the new e2e against the REAL `GENERATED_FILES_BY_TARGET`
registry (rather than each adapter package's own isolated unit tests)
surfaced two registry/reality mismatches that had gone undetected
because nothing previously cross-checked the registry's declared paths
against what `generateWorkspaceAdapters`'s dispatch actually writes:

1. **`copilot-vscode` never wrote its own declared files.**
   `GENERATED_FILES_BY_TARGET['copilot-vscode']` declares
   `.vscode/copilot-instructions.md` + `.vscode/settings.json`, but
   `workspace-adapters.ts`'s dispatch called `writeCopilotInstructions`
   without a `targetPath` override, so it silently fell back to that
   function's DEFAULT path — `.github/copilot-instructions.md` (shared,
   by coincidence, with the UNRELATED `github-copilot` target's own
   dedicated generator). `.vscode/settings.json` was never written at
   all by any code path for this target. Fixed: the `copilot-vscode`
   case now passes an explicit `targetPath` override
   (`.vscode/copilot-instructions.md`) and additionally writes a
   minimal `.vscode/settings.json` (`{}`). Scoped strictly to the
   `workspace-adapters.ts` multi-repo path — `configure.ts`'s SEPARATE
   `runConfigure` flow deliberately writes to
   `.github/copilot-instructions.md` for BOTH `copilot-vscode` and
   `github-copilot` (its own prior "B4 wiring" increment, with its own
   passing tests) and was NOT touched.
2. **`cursor`'s registry entry pointed at a file nothing ever wrote.**
   `GENERATED_FILES_BY_TARGET['cursor']` declared
   `.cursor/rules/axiom.mdc` — a path with zero references anywhere
   else in the codebase. The REAL, tested `generateCursorConfig`
   (0031/D.1) has always written `.cursor/settings.json` +
   `.cursor/AGENTS.md` (confirmed by the adapter package's own README
   and `generator.test.ts`). Fixed the registry entry to match the
   real generator's output (a metadata-only change — `cursor`'s actual
   generator was never touched, since it was already correct).

## Non-goals

- No fabricated per-slot native routing capability for claude-code,
  vscode, antigravity, or visual-studio-2026 — only genuinely verified
  native schemas are wired to auto-write.
- No new native MCP target added (antigravity / visual-studio-2026 /
  litellm remain unsupported in `NATIVE_MCP_TARGETS`, unchanged from
  `INC-20260708-mcp-native-config-mapping`).
- No auto-write to antigravity's user-global
  `~/.gemini/config/mcp_config.json` (isolation preserved, no opt-in
  merge command added in this increment — noted as a documented future
  consideration, not implemented, to keep this increment's diff
  focused).
- No new `vscode`-vs-`copilot-vscode` target split — this codebase's
  `ADAPTER_TARGETS`/`AdapterTargetId` enum has no separate `vscode`
  target; `copilot-vscode` IS the VS Code target family (confirmed by
  reading `init.ts`, `registry.ts`, `support-matrix.ts`, which lists
  BOTH `copilot-vscode` and a distinct `vscode` MVP entry with
  IDENTICAL `'fallback-only'` support level and no adapter package —
  `vscode` has no `packages/adapters/vscode` generator wiring anywhere
  in `workspace-adapters.ts`'s dispatch and is out of scope here; only
  `copilot-vscode`, which has a real generator, is deepened).
- No changes to `00`–`08` canonical spec docs beyond what's described
  under "General spec integration" below.
- No git commit (per orchestrator instructions).

## Acceptance criteria

- [x] `projectToClaudeCode` exists, mirrors `projectToOpencode`'s
      contract (no-op for wrong target, `Result`-based errors, atomic
      write), and is wired into `axiom model validate` as an optional,
      best-effort projection.
- [x] `@axiom/adapters-claude-code` exposes a `routing-snapshot.ts`
      loader mirroring the opencode adapter's, reading
      `.claude/model-routing.json`.
- [x] antigravity and visual-studio-2026 canonical files gain a
      target-specific note (MCP isolation / native-MCP-unverified
      respectively) without any auto-write to a shared/global file and
      without inventing an unverified native MCP schema.
- [x] New `apps/cli/tests/e2e/adapters.e2e.test.ts` asserts the full
      generated file set (per `GENERATED_FILES_BY_TARGET`) for all 8
      adapter targets in one hermetic in-process run, plus the two
      routing-snapshot projections' 7-slot shape.
- [x] (found while building the e2e, fixed) `copilot-vscode` actually
      writes its two declared files at their declared paths;
      `cursor`'s registry entry matches what its real generator writes.
- [x] `npm run build` clean.
- [x] `npx vitest run packages/adapters packages/model-routing
      packages/installer apps/cli` passes.
- [x] `npx vitest run apps/cli/tests/e2e` passes (new adapter e2e +
      existing MCP e2e).
- [x] `npx vitest run --no-file-parallelism` full run: 0 regressions
      vs the pre-increment baseline.

## Open questions

None blocking — the claude-code per-slot ask was resolved by
substituting a diagnostic snapshot for fabricated native capability
(see "Critical finding" above); visual-studio-2026 native MCP was
resolved by leaving it unimplemented with an honest in-file note
rather than inventing a schema.

## Assumptions

- "vscode" in the parent brief refers to this codebase's
  `copilot-vscode` target (the only VS-Code-family target with a real
  generator); the separate `vscode` entry in `SUPPORT_MATRIX`/
  `MVP_TARGETS` has no adapter package and is untouched.
- The claude-code routing snapshot is diagnostic/informational only in
  this increment — it is not read back into `generateClaudeCodeConfig`
  output. A future increment could wire
  `getSnapshotModelClass`-equivalent enrichment into the claude-code
  `## Routing note` block; not required by this increment's acceptance
  criteria.
- `writeThinCanonicalAgentsMd`'s new `extraNote` parameter is additive
  and optional; omitting it preserves the exact prior byte output for
  any caller that doesn't pass it (none currently do outside
  `workspace-adapters.ts`).
- The adapter e2e uses a single-repo (control-only) workspace spec
  (simpler than the existing 4-repo MCP e2e) because its purpose is
  breadth-per-adapter, not multi-repo propagation (already proven by
  the existing e2e for opencode/claude-code specifically).

## Implementation notes

- `packages/model-routing/src/claude-code-projection.ts`: copy-adapted
  from `opencode-projection.ts` — same 6-step flow (no-op check, load
  policy, load assignments, resolve all slots, mkdir `.claude/`, write
  JSON). Guard: `if (args.target !== 'claude-code') return { ok: true,
  value: { filePath: '', slotCount: 0, fellBackCount: 0 } };`. Exported
  as `projectToClaudeCode`, `ProjectToClaudeCodeArgs`,
  `ClaudeCodeProjection` from the package barrel.
- `packages/model-routing/src/index.ts`: added the three new exports
  next to the existing `projectToOpencode` export block, comment
  updated to `// C3c + this increment: doctor checks + opencode/
  claude-code projections`.
- `apps/cli/src/commands/model.ts` (`runModelValidate`): added a
  second `if (target === 'claude-code')` block, structurally identical
  to the opencode block, calling `projectToClaudeCode` and pushing the
  analogous formatted lines.
- `packages/adapters/claude-code/src/routing-snapshot.ts`: byte-for-
  byte structural port of `packages/adapters/opencode/src/
  routing-snapshot.ts` with the path constant changed to
  `path.join('.claude', 'model-routing.json')` and doc comments
  updated to reference claude-code/single-mode instead of opencode/
  multi-mode. Exported from `packages/adapters/claude-code/src/
  index.ts`.
- `apps/cli/src/commands/workspace-adapters.ts`
  (`writeThinCanonicalAgentsMd`): added an optional 6th parameter
  `extraNote?: string`; when present, appended as its own paragraph
  after the `TEAM:CUSTOM` block markers (outside the AXIOM:GENERATED
  region, since it's operator-facing guidance, not portable-entry
  content). The two call sites (`antigravity`, `visual-studio-2026`
  cases in `dispatchAdapterGenerator`) now pass a target-specific
  constant string:
  - antigravity: notes MCP servers are NOT auto-written to
    `~/.gemini/config/mcp_config.json` (user-global, shared across
    projects) and that adding Axiom's servers there is a manual,
    opt-in step outside this tool's default behavior.
  - visual-studio-2026: notes no verified native MCP schema exists for
    this target in this codebase yet; if `copilot-vscode` or
    `github-copilot` is also selected, `.vscode/mcp.json` may be
    readable by Visual Studio 2026 per its documented MCP config
    search paths, but this was not independently re-verified here.
- New/updated test files: `packages/model-routing/tests/
  claude-code-projection.test.ts` (mirrors `opencode-projection.test.ts`
  structure), `packages/adapters/claude-code/tests/
  routing-snapshot.test.ts` (mirrors the opencode adapter's own test),
  an addition to the existing `apps/cli/tests/model.test.ts` (test 12)
  for the new `runModelValidate` claude-code branch, and
  `apps/cli/tests/workspace-adapters.test.ts` additions for the
  `extraNote` parameter and the corrected copilot-vscode path/
  `.vscode/settings.json` assertions.
- `apps/cli/tests/e2e/adapters.e2e.test.ts` (new): single-repo
  workspace, all 8 `ADAPTER_TARGETS`, asserts `GENERATED_FILES_BY_
  TARGET` coverage + marker shape + (after calling `runModelValidate`
  twice, once per target override `opencode`/`claude-code`) both
  routing-snapshot files' 7-slot shape.

## Validation

### `npm run build` (tsc -b), from `Axiom/`

```
> axiom-product@0.1.0 build
> tsc -b

(clean exit, no errors)
```

### `npx vitest run packages/adapters packages/model-routing packages/installer apps/cli`

```
 Test Files  82 passed (82)
      Tests  726 passed (726)
```

Includes this increment's new suites:
`packages/model-routing/tests/claude-code-projection.test.ts` (6),
`packages/adapters/claude-code/tests/routing-snapshot.test.ts` (7),
`apps/cli/tests/model.test.ts` addition (test 12, claude-code
diagnostic projection), `apps/cli/tests/workspace-adapters.test.ts`
additions (antigravity/visual-studio-2026 extra-note assertions +
corrected copilot-vscode path/settings.json assertions).

### `npx vitest run apps/cli/tests/e2e`

```
 Test Files  2 passed (2)
      Tests  7 passed (7)
```

New `adapters.e2e.test.ts` (3 tests: full 8-target file-set coverage
including the copilot-vscode/cursor path fixes below, marker-shape
assertions scoped to the targets that actually use the
`AXIOM:GENERATED` convention, both routing-snapshot 7-slot assertions)
+ existing `workspace-mcp.e2e.test.ts` (4 tests, unchanged, still
green).

### `npx vitest run --no-file-parallelism` (full repo)

```
 Test Files  192 passed (192)
      Tests  1995 passed (1995)
```

0 regressions vs the 1978/1978 baseline; net +17 tests (6 + 7 + 1 + 3
from the suites listed above), matching exactly.

## Result

Implemented as scoped, with one deliberate, documented deviation from
the parent brief: claude-code does NOT get fabricated per-slot native
routing (the codebase's own locked `SUPPORT_MATRIX`/`resolver`/
`fallback`/`checks` contract makes claude-code's single-mode behavior
a tested invariant, not a gap) — instead it gets an honest diagnostic
routing-snapshot projection (`projectToClaudeCode` ->
`.claude/model-routing.json`) that surfaces the real single-mode
resolution (uniform `medium` + `fallbackReason:
'per-slot-routing-unsupported'` across all 7 slots), plus a matching
adapter-side loader (`@axiom/adapters-claude-code/routing-snapshot`)
mirroring opencode's. antigravity and visual-studio-2026 gain
target-specific operator-facing notes in their canonical AGENTS.md/
AXIOM.md (MCP-isolation guidance and native-MCP-unverified guidance,
respectively) without any auto-write to a shared/global file and
without inventing an unverified schema. vscode/copilot-vscode already
had generator + native-MCP wiring from prior increments, but building
the e2e surfaced that its OWN declared files were never actually being
written at their declared paths — fixed (see below). A new hermetic
`apps/cli/tests/e2e/adapters.e2e.test.ts` proves the full
`GENERATED_FILES_BY_TARGET` file set materializes correctly for all 8
adapter targets in one workspace, plus the 7-slot shape of both
routing snapshots; building it caught two genuine pre-existing
registry/reality mismatches, both fixed in this increment:
`copilot-vscode` was silently writing to `.github/copilot-
instructions.md` (a different target's default path) instead of its
own declared `.vscode/copilot-instructions.md`, and never wrote
`.vscode/settings.json` at all; `cursor`'s registry entry declared a
path (`.cursor/rules/axiom.mdc`) that no generator has ever written,
while the real generator's actual output
(`.cursor/settings.json`/`.cursor/AGENTS.md`) went undeclared. Both
fixes are scoped narrowly (one dispatch case + one registry line) and
do not touch the underlying, already-correct adapter generators
themselves. All acceptance criteria met; build + targeted suites + e2e
+ full regression suite green (1995/1995, +17 vs the 1978 baseline).

## General spec integration

Integrated into the canonical spec
(`Axiom.Spec/specs/06_Integraciones_y_Capacidades.md`) — appended a
new subsection **"Adapter depth: snapshot diagnóstico de claude-code y
notas por-target (antigravity / visual-studio-2026)"** right after the
existing "Targets soportados hoy" support-matrix table, citing
`INC-20260708-adapters-depth`, stating: (1) claude-code's `single-mode`
is an intentional, tested resolver contract, NOT a gap —
`.claude/model-routing.json` is a diagnostic mirror of
`.opencode/model-routing.json`, not a native routing input; (2)
antigravity's MCP servers are never auto-written to its user-global
config, by design, with a manual opt-in path documented in its
generated AGENTS.md instead; (3) visual-studio-2026's native MCP
schema remains unverified — no file is invented; its generated
AXIOM.md carries an informational note only; (4) the two genuine
registry/reality mismatches found and fixed while building the e2e
(`copilot-vscode`'s files never landing at their declared paths;
`cursor`'s registry entry pointing at a path nothing wrote). No
changes to 00–05, 07, 08 (no contradiction found — this increment's
knowledge is additive to 06's existing adapter/model-routing
sections).

**Reconciliation by the batch's final consolidation pass**: the 06
subsection this increment already appended
("Adapter depth: snapshot diagnóstico de claude-code y notas
por-target") was left intact (not duplicated). The final pass
additionally cited this increment in **02_Requisitos_No_Funcionales.md**
(NFR-AXM-010 e2e note) for the new `adapters.e2e.test.ts` that
materializes all 8 `ADAPTER_TARGETS` and verifies
`GENERATED_FILES_BY_TARGET` per target. No other 00-08 file needed a
change for this increment.
