# Increment: adapter agent tuning (verbosity/personality preamble)

Status: closed
Date: 2026-07-15

## Goal

Adapter configurations carry per-adapter "agent tuning" settings (verbosity,
personality, optional model hint) so the pre-generated prompt tells the agent
to be concise, pragmatic, and token-economical — work faster, spend fewer
tokens, stay focused on the task. The tuning flows from the adapter routing
table into the crafted prompt and is visible in the launcher UI.

## Context

`@axiom/launcher`'s `craftPrompt` (in `adapter-routing.ts`) already resolves,
per selected adapter, a header/mention line and a launch mechanism, then hands
off to the adapter-neutral `buildPrompt` (`prompt-builder.ts`). There was no
concept of per-adapter prompt-shaping hints (verbosity/personality/model).
The launcher's server layer (`apps/cli/src/commands/app-launcher.ts`) and its
static front (`apps/cli/static/launcher/launcher.js`) only surfaced
`id`/`promptDialect`/`launchMechanism` per adapter.

## Scope

- New `AgentTuning` type (`model?`, `verbosity?: 'low'|'medium'|'high'`,
  `personality?: 'pragmatic'|'balanced'|'thorough'`) in
  `packages/launcher/src/types.ts`, exported from `packages/launcher/src/index.ts`.
- `PromptBuildOptions.agentTuning?: AgentTuning` (optional; absent → no
  preamble emitted).
- `AdapterRoutingEntry.agentTuning?: AgentTuning` in `adapter-routing.ts`. All
  three shipped adapters (`claude-code`, `github-copilot`, `cli`) carry the
  SAME default: `{ verbosity: 'low', personality: 'pragmatic' }` (no `model`
  default).
- `craftPrompt` resolves the adapter entry and passes its `agentTuning` into
  `buildPrompt`.
- `buildPrompt` (`prompt-builder.ts`) renders a new deterministic "Ajustes del
  agente" preamble block, inserted immediately after the header block and
  before the structured-values block, via a new pure helper
  `buildAgentTuningLines(tuning): string[]`. Emitted only when at least one
  tuning field is set.
- `apiGetLauncherData` (`apps/cli/src/commands/app-launcher.ts`) adds
  `agentTuning: a.agentTuning ?? null` to each adapter entry in the response,
  and the `LauncherDataResponse.adapters` element type gains
  `agentTuning?: { model?: string; verbosity?: string; personality?: string } | null`.
- The static front (`apps/cli/static/launcher/launcher.js`) appends a
  compact `verbosity/personality` label to each adapter `<option>` in
  `renderControls` (read-only, no new controls).
- New focused test file `packages/launcher/tests/agent-tuning.test.ts`.
- Regenerated `packages/launcher/tests/__snapshots__/adapter-routing-snapshot.test.ts.snap`
  to include the new preamble (same tuning on all 3 shipped adapters keeps
  the adapter-neutral-body snapshot invariant intact — the preamble content
  is identical across adapters, only NEW).

## Non-goals

- Do not wire `verbosity`/`personality`/`model` into model-routing or
  provider selection — these are prompt-shaping hints only.
- Do not touch `@axiom/tui` (kept generic).
- Do not add asset files; all canonical content stays as TS string
  constants/functions.
- Do not give different shipped adapters different default tuning (all three
  intentionally share the same default in this increment).

## Acceptance criteria

- [x] `AgentTuning` type exists in `packages/launcher/src/types.ts` and is
      exported from `packages/launcher/src/index.ts`.
- [x] All three shipped adapters (`claude-code`, `github-copilot`, `cli`) in
      `AXIOM_ADAPTER_ROUTING` carry `agentTuning: { verbosity: 'low',
      personality: 'pragmatic' }`.
- [x] A prompt crafted via `craftPrompt` with the default table contains the
      "Ajustes del agente:" preamble with `- verbosidad: low` and
      `- estilo: pragmatic` lines plus the closing directive sentence.
- [x] `buildPrompt` called with no `agentTuning` emits no preamble block.
- [x] The launcher server data (`GET /launcher/data`) and the static front
      (`renderControls`) surface the tuning per adapter.
- [x] `npm run build` (tsc -b) passes clean.
- [x] `packages/launcher` vitest suite is green (snapshots regenerated once,
      then confirmed stable on a second run without `-u`).
- [x] Relevant `apps/cli` launcher tests are green (narrowed run, see
      Validation).

## Open questions

none — resolved by orchestrator

## Assumptions

- Same tuning on all three shipped adapters is intentional per the brief
  (keeps the per-adapter prompt-body snapshot invariant provable: identical
  tuning → identical preamble across adapters).
- `model` is left unset by default on all three adapters (only a documented
  optional override point for projects that ship their own routing table).

## Implementation notes

Files changed (`Axiom` product repo):

- `packages/launcher/src/types.ts` — added `AgentTuning`; added
  `agentTuning?: AgentTuning` to `PromptBuildOptions`.
- `packages/launcher/src/index.ts` — exported `AgentTuning`.
- `packages/launcher/src/adapter-routing.ts` — added
  `AdapterRoutingEntry.agentTuning?: AgentTuning`; set the same default
  tuning on all three shipped adapters; `craftPrompt` now resolves the
  adapter entry (`resolveAdapter(table, adapterId)`) and passes
  `agentTuning: entry?.agentTuning` into `buildPrompt`.
- `packages/launcher/src/prompt-builder.ts` — added pure helper
  `buildAgentTuningLines(tuning): string[]` (returns `[]` when no field is
  set); `buildPrompt` inserts the rendered block as a new entry in `blocks`
  right after the header block, only when `options.agentTuning` yields
  non-empty lines.
- `packages/launcher/tests/agent-tuning.test.ts` — new; asserts (a) default
  table + `craftPrompt` contains the preamble with `low`/`pragmatic`, (b) no
  `agentTuning` → no preamble, (c) two custom adapter entries with different
  tuning produce different prompt bodies.
- `packages/launcher/tests/__snapshots__/adapter-routing-snapshot.test.ts.snap`
  — regenerated (adds the identical preamble block to all three per-adapter
  snapshots; adapter-neutral-body invariant still holds since the tuning is
  the same across adapters).
- `apps/cli/src/commands/app-launcher.ts` — `LauncherDataResponse.adapters`
  element type gains `agentTuning`; `apiGetLauncherData` maps
  `agentTuning: a.agentTuning ?? null` per adapter.
- `apps/cli/static/launcher/launcher.js` — `renderControls` appends a
  `verbosity/personality` label to each adapter `<option>` text when
  `a.agentTuning` is present.

## Validation

Run from `C:/repos/Axiom Workspace/Axiom`:

1. `npm run build` (tsc -b) — clean, no errors.
2. `npx vitest run -u packages/launcher` — regenerated 3 snapshots
   (`claude-code`, `github-copilot`, `cli` under the A1 describe block); all
   6 test files / 36 tests passed.
3. `npx vitest run packages/launcher` (no `-u`) — confirmed green, same 6
   files / 36 tests, no snapshot diffs.
4. `npx vitest run apps/cli/tests -t launcher` — 4 test files matched
   (`app-launcher.test.ts`, `launcher-push.test.ts`,
   `launcher-front-no-vscode.test.ts`, `launcher-panels.test.ts`), 34 tests
   passed, 0 failed (the rest of the `apps/cli` suite was skipped by the
   `-t launcher` title filter, as expected — not run in this increment to
   keep validation focused and fast).

No PRE-EXISTING nor NEWLY-INTRODUCED failures were observed in any of the
above runs.

## Result

Implemented `AgentTuning` end-to-end: type + export, default tuning on all
three shipped adapters, `craftPrompt` → `buildPrompt` wiring, a new
deterministic "Ajustes del agente" preamble block (only emitted when tuning
is present), and surfaced the tuning in both the launcher server response and
the static front's adapter selector. Added a focused test file covering the
present/absent/differing-tuning cases. Regenerated and re-verified the
per-adapter prompt snapshots — the adapter-neutral-body invariant
(`adapter-routing-snapshot.test.ts`) still holds because all three shipped
adapters share the identical tuning, so the new preamble is identical across
adapters and only the header/mention line differs, as before. Build and all
targeted tests are green.

## General spec integration

Integrado por el pase final de la orquestación (autopilot, 2026-07-15):
- `01_Requisitos_Funcionales.md` → **RF-AXM-029 Tuning de agente por adapter**.
- `02_Requisitos_No_Funcionales.md` → **NFR-AXM-016 Economía de tokens y foco del agente**.
- `05_Interfaces_Operativas.md` → sección "tanda INC-20260715-*" (selector de adapter muestra el tuning; prompt pregenerado con preámbulo).
- `06_Integraciones_y_Capacidades.md` → sección "tanda INC-20260715-*" (tuning `agentTuning` por adapter, invariante del snapshot preservada).
- `08_Glosario.md` → término "Tuning de agente (`agentTuning`)".
- `00_Resumen_Ejecutivo.md` → mención en la tanda (prompt pregenerado por adapter con tuning `low`/`pragmatic`).

Archivado en `Axiom.Spec/specs/increments/_archive/`.
