# Increment: Adapter target registry — canonical alignment (codex + vscode)

Status: closed
Date: 2026-07-26

## Goal

Make the adapter target registry a single source of truth and align it
across every layer that declares or consumes the list of adapter target
ids: the CLI target list (`apps/cli/src/commands/init.ts`'s
`ADAPTER_TARGETS`), the install-profiles composer's allowed-target
registry (`packages/install-profiles/src/default-profiles.ts`), the
`axiom.yaml` `capabilities.adapters` headline list, and the model-routing
support matrix (`packages/model-routing/src/support-matrix.ts`). Add
`codex` as a NEW recognized adapter target and add `vscode` as an
explicit target (it already had a support-matrix entry and an adapter
package, `@axiom/adapters-vscode`, but was missing from the CLI target
list). Real, rich generators/MCP/skills for `codex` come in a later
increment (INC-2/INC-3, deferred by design) — this increment only
registers the ids and gives both new targets the SAME safe thin/fallback
treatment that `antigravity`/`visual-studio-2026` already get, so nothing
breaks.

## Context

The adapter target vocabulary was declared independently in several
places that had drifted:

- `apps/cli/src/commands/init.ts`'s `ADAPTER_TARGETS` (8 ids) — the CLI's
  own source of truth, re-declared inline 3 more times (2 TS unions +
  1 `validTargets` const for `--target` parsing).
- `packages/install-profiles/src/default-profiles.ts`'s `DEFAULT_PROFILES`
  — a SEPARATE 8-id list (`adapterTargets` + both profiles'
  `allowedTargets`) that the composer (`resolveInstallProfile`) validates
  against; its own header comment already warned "MUST list all 8 targets
  ... or `configure`/`start` will break".
- `packages/model-routing/src/support-matrix.ts`'s `SUPPORT_MATRIX` /
  `MVP_TARGETS` — a 9-id list (includes `vscode`, which the other two
  registries did NOT have).
- `axiom.yaml`'s `capabilities.adapters` — a narrower 5-id "headline" list.
- `apps/cli/src/commands/tui.ts`'s `TARGET_LABELS` — an exhaustive
  `Record<AdapterTarget, string>` keyed off `ADAPTER_TARGETS`.
- `apps/cli/src/commands/workspace-adapters.ts`'s per-target dispatch
  `switch` — an exhaustive switch (`const exhaustiveCheck: never = target`)
  that fails to compile if `ADAPTER_TARGETS` gains a member without a
  matching `case`.
- `packages/installer/src/registry.ts`'s `GENERATED_FILES_BY_TARGET` — one
  entry per target, consumed by an e2e test
  (`apps/cli/tests/e2e/adapters.e2e.test.ts`) that iterates ALL
  `ADAPTER_TARGETS` and asserts every declared file exists and is
  non-empty after `generateWorkspaceAdapters` runs.

`vscode` already had a real generator (`@axiom/adapters-vscode`,
`generateVscodeConfig`, already wired into `apps/cli/src/commands/sync.ts`)
and an entry in `SUPPORT_MATRIX`/`MVP_TARGETS`, but was invisible to
`init`/`workspace-adapters`/`default-profiles` — a project could never
select it via `axiom init --target vscode` or the workspace-setup wizard.
`codex` did not exist anywhere in the codebase as a recognized id.

## Scope

- `apps/cli/src/commands/init.ts`: `ADAPTER_TARGETS` (canonical array,
  now 10 ids), `InitArgs.target`/`ProfileTriple.adapterTarget` unions, the
  `--target` help string, and the `validTargets` const used to parse
  `--target` — all extended with `'vscode'` and `'codex'`, appended at the
  END of every list (never inserted mid-list, to avoid shifting any
  index-based test fixture). `DEFAULT_TARGET` stays `'opencode'`.
- `apps/cli/src/commands/tui.ts`: `TARGET_LABELS` (an exhaustive
  `Record<(typeof ADAPTER_TARGETS)[number], string>`) gains `vscode`/
  `codex` entries — this was a compile-breaking site not called out by
  name in the original brief, found by grepping for every place that
  indexes an object literal by the full `AdapterTarget` union.
- `axiom.yaml`: `capabilities.adapters` becomes the headline set
  `claude-code, github-copilot, vscode, opencode, cursor, antigravity,
  visual-studio-2026, codex` (per explicit instruction). `copilot-vscode`
  and `litellm` remain recognized internal targets but are intentionally
  left out of this headline list.
- `packages/model-routing/src/support-matrix.ts`: `codex` added to
  `SUPPORT_MATRIX` (`'fallback-only'`) and `MVP_TARGETS` (now 10 entries);
  `vscode` was already present in both. Header comments updated
  (9 → 10 targets).
- `packages/model-routing/tests/support-matrix.test.ts`: counts (9 → 10)
  and groupings updated; `codex` moved out of the "non-MVP" assertion
  group into its own "declares the codex target" test, plus an
  `isMvpTarget('codex') === true` assertion.
- `apps/cli/src/commands/workspace-adapters.ts`: two new `case`s in the
  per-target dispatch `switch` (required by the exhaustive
  `exhaustiveCheck: never = target` guard already present in this file):
  - `case 'vscode'`: wired to the REAL generator,
    `@axiom/adapters-vscode`'s `generateVscodeConfig` (writes
    `.vscode/settings.json` + `.vscode/extensions.json`), NOT the thin
    canonical path — `vscode` already had a dedicated generator package,
    it was only missing the dispatch wiring.
  - `case 'codex'`: routed through the existing
    `writeThinCanonicalAgentsMd` helper (same treatment as `antigravity`/
    `visual-studio-2026`), writing `.codex/AGENTS.md` with a target-specific
    extra note (`CODEX_EXTRA_NOTE`) explaining explicitly that `codex` has
    no dedicated generator/MCP/skills yet and that a later increment
    completes real support.
- `packages/installer/src/registry.ts`: `GENERATED_FILES_BY_TARGET` gains
  `'vscode': ['.vscode/settings.json', '.vscode/extensions.json']` and
  `'codex': ['.codex/AGENTS.md']`.
- `packages/install-profiles/src/default-profiles.ts` +
  `packages/install-profiles/tests/default-profiles.test.ts` (SCOPE
  ADDITION beyond the literal edit list — see Decisions): `vscode`/`codex`
  added to `DEFAULT_PROFILES.adapterTargets` (`mvp: false`, matching
  `cursor`/`github-copilot`) and to BOTH profiles'
  `profileBindings.*.allowedTargets`; the test's local hard-coded
  8-id `ADAPTER_TARGETS` fixture extended to 10 and its parameterized
  `resolveInstallProfile` coverage now also proves `codex`/`vscode` for
  both `(builder, product-owner) × standard` combinations.
- Test additions/updates (existing hardcoded lists, per the brief's
  instruction to grep for and update them): `apps/cli/tests/
  workspace-adapters.test.ts` (2 new focused tests: `vscode`'s real output,
  `codex`'s thin canonical output), `apps/cli/tests/e2e/
  adapters.e2e.test.ts` (8 → 10 targets in title/comments, `codex` added
  to `CANONICAL_MARKER_TARGETS`, 2 new target-specific assertions mirroring
  the existing antigravity/vs2026 ones).

## Non-goals

- Real generators, native MCP config projection, or skills for `codex` —
  explicitly deferred to a later increment (INC-2/INC-3). `codex` gets
  ONLY the thin canonical `AGENTS.md` treatment in this increment.
- Adding `codex`/`vscode` to `NATIVE_MCP_TARGETS`
  (`apps/cli/src/commands/native-mcp-config.ts`) — that dispatcher's
  `switch` already has a non-exhaustive `default:` branch that degrades
  to a warning for any target without a native schema, so `codex`/`vscode`
  already fall through it safely with zero code change; adding them there
  is explicitly out of scope for this increment (INC-3 per the brief).
- Touching `apps/cli/src/commands/sync.ts` — it already has a working
  `case 'vscode'` (calls `generateVscodeConfig`) and its `materializeAdapterOutputs`
  switch is NOT exhaustive (plain `adapterTarget: string` + a `default:`
  no-op), so `codex` already falls through it exactly like `antigravity`/
  `visual-studio-2026` do today (zero-file no-op) — no change needed or
  made.
- Renumbering/reordering any existing target's position in any of the
  canonical arrays — every new id was APPENDED at the end everywhere, to
  avoid invalidating index-based test fixtures (e.g.
  `apps/cli/tests/tui.test.ts`'s "índice 2 = claude-code" comment).
- Any change to `@axiom/adapters-vscode`'s own generator, settings
  renderer, or its own package tests — used as-is.

## Acceptance criteria

- [x] Canonical `AdapterTarget` union / `ADAPTER_TARGETS` is exactly:
      `opencode, copilot-vscode, claude-code, antigravity,
      visual-studio-2026, cursor, github-copilot, litellm, vscode, codex`
      (10 ids); `DEFAULT_TARGET` stays `'opencode'`.
- [x] `axiom.yaml`'s `capabilities.adapters` lists the headline set:
      `claude-code, github-copilot, vscode, opencode, cursor, antigravity,
      visual-studio-2026, codex`.
- [x] `SUPPORT_MATRIX`/`MVP_TARGETS` include `codex` as `'fallback-only'`
      (10 entries total); `vscode` was already present.
- [x] Every exhaustive switch/record keyed off the full `AdapterTarget`
      union (`workspace-adapters.ts`'s dispatch switch, `tui.ts`'s
      `TARGET_LABELS`) compiles clean with `codex`/`vscode` handled.
- [x] `npm run build` (`tsc -b`) passes clean.
- [x] `codex`/`vscode` get a working, non-throwing generation path through
      `generateWorkspaceAdapters` even when selected ALONE (i.e. as the
      PRIMARY/first adapter, which drives the one `installProfile` call
      per repo) — proven by 2 new focused unit tests and by the existing
      8→10-target e2e test.
- [x] Every test that hardcoded the previous 8/9-target list or count was
      found (via targeted grep) and updated to the new 10-target set.
- [x] Targeted test files pass (see Validation).

## Decisions

1. **Scope was extended, with justification, to
   `packages/install-profiles/src/default-profiles.ts` +
   its test**, even though the brief's "exact edit surface" did not name
   this file. Reason: `resolveInstallProfile` (the install-profiles
   composer) THROWS if `adapterTarget` is not in
   `profileBindings[profile].allowedTargets`; `generateWorkspaceAdapters`
   calls `installProfile` ONCE per repo using `args.adapters[0]` as the
   adapter target, wrapped in try/catch — so if `vscode`/`codex` were
   picked as the PRIMARY (first) adapter without being in
   `allowedTargets`, the ENTIRE per-repo adapter generation would degrade
   to a single warning and write ZERO files for ANY target in that repo,
   not just the new one. That is NOT the "same safe thin/fallback
   treatment antigravity/visual-studio-2026 currently get" the brief asks
   for (those two already validate fine as sole/primary adapters — see
   `apps/cli/tests/workspace-adapters.test.ts`'s pre-existing
   `visual-studio-2026`-only test). `default-profiles.ts`'s own header
   comment already explicitly warns future maintainers about exactly this
   reconciliation requirement. Doctor's `IP-003` check (`packages/doctor/
   src/checks.ts`) — "every adapterTarget in allowedTargets must be in
   `MVP_TARGETS`" — independently confirms this is safe: `vscode` was
   already in `MVP_TARGETS`, and `codex` is added to it in this same
   increment, so `IP-003` keeps passing.
   `packages/install-profiles/tests/default-profiles.test.ts` was ALSO
   found by the brief's own instruction #5 ("grep for any test that
   hardcodes the adapter-target count or list") — it hard-codes an 8-id
   local `ADAPTER_TARGETS` const and parameterizes `resolveInstallProfile`
   success across it, so it was already squarely in scope, not a
   tangential addition.
2. **`vscode` is wired to its REAL generator (`@axiom/adapters-vscode`'s
   `generateVscodeConfig`), not the thin canonical path**, per the
   brief's explicit instruction ("if a `generateVscodeConfig` export
   exists, wire the vscode case to it instead of thin"). It writes
   `.vscode/settings.json` (`{}`, MVP) + `.vscode/extensions.json`
   (`{recommendations: [], unwantedRecommendations: []}`), no `extensions`
   input wired yet (no caller of `generateWorkspaceAdapters` collects
   one) — matches the pre-existing precedent in `sync.ts`'s own
   `case 'vscode'`.
3. **`codex` gets the thin canonical `AGENTS.md` treatment** (`.codex/
   AGENTS.md`, via `writeThinCanonicalAgentsMd`), with an explicit
   target-specific note (`CODEX_EXTRA_NOTE`) stating it has no
   dedicated generator/MCP/skills yet — mirrors `antigravity`/
   `visual-studio-2026`'s existing pattern exactly, including their own
   target-specific notes.
4. **`vscode`/`codex` both get `mvp: false`** in
   `DEFAULT_PROFILES.adapterTargets` — `vscode` because, despite having a
   real generator, it is (like `cursor`/`github-copilot`) not one of the
   original "MVP 5"; `codex` because it only has thin/fallback treatment
   so far.
5. **New ids were always APPENDED, never inserted mid-list**, in every
   canonical array (`ADAPTER_TARGETS`, `validTargets`, `SUPPORT_MATRIX`,
   `MVP_TARGETS`, `DEFAULT_PROFILES.adapterTargets`/`allowedTargets`),
   specifically to avoid shifting the numeric indices some existing tests
   depend on (e.g. `apps/cli/tests/tui.test.ts`'s adapters multi-select
   scripts a literal index like `'2'` for "claude-code").
6. **`native-mcp-config.ts` and `sync.ts` were left untouched**, confirmed
   safe rather than assumed safe: both already degrade new/unknown
   targets to a warning/no-op via a non-exhaustive `switch`/`default:`
   branch (no `never`-typed exhaustiveness check in either file), so
   `codex`/`vscode` already fall through them exactly like `antigravity`/
   `visual-studio-2026` do today.

## Validation

Ran from `C:\repos\Axiom Workspace\Axiom`:

- `npm run build` (`tsc -b`) — **PASSED**, clean, exit code 0 (run twice:
  once right after the source edits, once again as part of the final
  review pass).
- Targeted tests —
  `npx vitest run packages/model-routing apps/cli/tests/native-mcp-config.test.ts
  apps/cli/tests/workspace-adapters.test.ts apps/cli/tests/e2e/adapters.e2e.test.ts
  packages/install-profiles apps/cli/tests/init.test.ts apps/cli/tests/tui.test.ts`
  — **23 files / 256 tests passed**, zero failures. Breakdown of the
  files this increment's changes directly touch:
  - `packages/model-routing/tests/support-matrix.test.ts`: 12/12 passed
    (updated for the 10-target set).
  - `apps/cli/tests/workspace-adapters.test.ts`: 8/8 passed (2 new:
    `vscode`, `codex`).
  - `apps/cli/tests/e2e/adapters.e2e.test.ts`: 3/3 passed (updated for
    10 targets).
  - `packages/install-profiles/tests/default-profiles.test.ts`: 25/25
    passed (extended fixture + parameterized coverage for `vscode`/
    `codex`).
  - `apps/cli/tests/native-mcp-config.test.ts`: 15/15 passed (confirmed
    UNCHANGED — `NATIVE_MCP_TARGETS` still exactly the pre-existing
    5 ids).
  - `apps/cli/tests/init.test.ts`, `apps/cli/tests/tui.test.ts`: 17/17
    and 30/30 passed (no index-based regression from appending the 2 new
    targets at the end of every list).
- `npx vitest run packages/installer packages/doctor` — **20 files /
  195 tests passed** (`packages/doctor/tests/adapters.test.ts`'s TC-009
  — "6 packages have `src/generator.ts` + `dist/index.js`" — is a
  hardcoded package-directory count unrelated to `ADAPTER_TARGETS`;
  `vscode`'s package was already counted among those 6; `codex` has no
  package yet by design, so this check needed no change and still
  passes).
- `npx vitest run packages` (full packages sweep, extra safety pass per
  the brief's "you may also run the full set") — **192 files / 2069
  tests passed**, zero failures, including
  `packages/adapters/vscode/tests/generator.test.ts` (9/9, confirmed
  untouched/still green).
- `npx vitest run apps/cli/tests` (full apps/cli suite) — **123 files
  passed, 2 files reported a single timeout failure each**
  (`context.test.ts`'s Scenario 2, `launcher-panels.test.ts`'s
  `POST /launcher/role-branch` confirmed test) — **1154 passed / 2
  failed** under full-parallel load. Re-ran BOTH failing files in
  isolation: **2 files / 26 tests, 100% passed** — confirming these are
  pre-existing flaky-under-parallel-load tests (consistent with the
  memory note from `INC-20260724-worktree-mode-selection`, which already
  flagged `context.test.ts`/`workspace-setup.test.ts` as
  flaky-under-parallel-load), NOT a regression introduced by this
  increment. No file this increment touches is among the ones that
  timed out.

No pre-existing failures were introduced by this increment's changes.

## Result

Implemented. The canonical `ADAPTER_TARGETS` is now 10 ids:
`opencode, copilot-vscode, claude-code, antigravity, visual-studio-2026,
cursor, github-copilot, litellm, vscode, codex` — reconciled across the
CLI (`init.ts`, `tui.ts`), `axiom.yaml`'s headline capability list, the
model-routing support matrix, the install-profiles composer's allowed-
target registry, and the installer's generated-files registry. `vscode`
(previously invisible to `init`/`workspace-adapters` despite already
having a real adapter package) is now wired end-to-end through its real
generator. `codex` (a brand-new id) is registered everywhere and gets the
same safe thin-canonical `AGENTS.md` fallback treatment `antigravity`/
`visual-studio-2026` already have, with an explicit note documenting that
richer support (generator, native MCP, skills) is deferred. Every
exhaustive switch/record keyed off the adapter-target union was found and
given explicit handling; the build stays clean and every targeted test
(plus a full `packages` sweep and a full `apps/cli` sweep, modulo 2
confirmed pre-existing flaky tests) passes.

## General spec integration

Not integrated into `Axiom.Spec/specs/00..08` directly — per this task's
instructions, the orchestrator performs the final integration pass across
the batch. Stable facts that should be integrated when that pass runs:

- The canonical adapter target vocabulary is now 10 ids (list above),
  with `codex` explicitly documented as thin/fallback-only (no generator/
  MCP/skills yet) and `vscode` explicitly distinct from `copilot-vscode`
  (own generator package, `@axiom/adapters-vscode`).
- The reconciliation invariant this increment restores/hardens: any
  future adapter target addition MUST touch (a) `apps/cli/src/commands/
  init.ts`'s `ADAPTER_TARGETS` + its 3 inline re-declarations, (b)
  `packages/install-profiles/src/default-profiles.ts`'s `adapterTargets`
  + BOTH profiles' `allowedTargets` (or `installProfile`/
  `generateWorkspaceAdapters` will silently reject that target as
  primary), (c) `packages/model-routing/src/support-matrix.ts`'s
  `SUPPORT_MATRIX`/`MVP_TARGETS`, (d) `apps/cli/src/commands/tui.ts`'s
  `TARGET_LABELS`, and (e) `apps/cli/src/commands/workspace-adapters.ts`'s
  exhaustive dispatch switch — all 5 are compile- or runtime-enforced
  today (the first via TS structural typing, the switch via an explicit
  `never`-typed exhaustiveness guard, the rest via tests) so a future
  increment cannot add a target and forget one silently.
- `codex`/INC-2/INC-3 follow-up: a real `@axiom/adapters-codex` generator,
  a `codex` entry in `NATIVE_MCP_TARGETS`, and codex-specific skills are
  explicitly deferred, not implemented.
