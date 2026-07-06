# Increment: Workspace adapter templates bundling

Status: closed
Date: 2026-07-06

## Goal

Make the workspace adapter generation produce REAL content for the
template-dependent adapters (opencode `.opencode/AGENTS.md`, claude-code
`.claude/AGENTS.md`, copilot-vscode `.github/copilot-instructions.md`) by
bundling the canonical instruction templates into the product and feeding
them to the generators, so a freshly-created workspace repo with these
adapters selected gets non-empty adapter instruction files and no
`template-missing` warnings.

## Context

`INC-20260705-workspace-adapters-multiselect` added
`apps/cli/src/commands/workspace-adapters.ts`'s `generateWorkspaceAdapters`,
called by `runWorkspaceSetup` to generate adapter files into every repo of
a workspace. It documented (as an explicit, accepted assumption, not a
bug) that the adapter instruction templates
(`agents-md-template.md`, `copilot-instructions.template.md`) are not
bundled anywhere in the product — they only exist in the sibling
`Axiom.Spec/templates/`. As a result, on freshly-scaffolded repos the
opencode/claude-code/copilot-vscode generators degrade to
`template-missing` warnings and do not write their instruction files (only
`.opencode/mcp.json` etc. get written). This increment closes that gap.

## Scope

- Bundle `Axiom.Spec/templates/agents-md-template.md` and
  `Axiom.Spec/templates/copilot-instructions.template.md` verbatim as TS
  string constants in a new module
  `apps/cli/src/commands/workspace-adapter-templates.ts`.
- Add an optional `templateContent?: string` parameter to
  `generateOpencodeConfig` (`@axiom/adapters-opencode`) and
  `generateClaudeCodeConfig` (`@axiom/adapters-claude-code`): when
  provided, it is used instead of reading `templatePath` from disk;
  omitted, behavior is byte-for-byte identical to today (reads
  `templatePath` / default path).
- `generateWorkspaceAdapters` passes the bundled `AGENTS_MD_TEMPLATE` as
  `templateContent` to opencode/claude-code, and the bundled
  `COPILOT_INSTRUCTIONS_TEMPLATE` as the `template` content to
  `writeCopilotInstructions` (copilot-vscode) — no filesystem template
  read for these three adapters anymore.
- Update `apps/cli/tests/workspace-adapters.test.ts`,
  `apps/cli/tests/workspace-setup.test.ts`,
  `apps/cli/tests/workspace-mcp.test.ts` to reflect that these three
  adapters now produce real, non-empty content with no `template-missing`
  warning.
- Add/extend unit tests for the two adapter generators covering
  `templateContent` provided vs. omitted.

## Non-goals

- No changes to `configure.ts`/`sync.ts`/`init.json` — the single-repo
  path keeps reading `templatePath` from disk exactly as before.
- No changes to the generators' default `templatePath` resolution.
- No change to `antigravity`/`visual-studio-2026` (canonical thin
  AGENTS.md via `renderCanonicalAgentsMd`) or `litellm`/`cursor`
  (no template dependency).
- No git operations.
- No integration into `Axiom.Spec/specs/00-08` in this pass (left to the
  orchestrator's cross-increment integration pass, matching sibling
  `INC-20260705-workspace-*` increments' precedent).

## Acceptance criteria

- [x] `AGENTS_MD_TEMPLATE`/`COPILOT_INSTRUCTIONS_TEMPLATE` bundled as TS
      constants, verbatim copies of the canonical `Axiom.Spec/templates/`
      files.
- [x] `generateOpencodeConfig`/`generateClaudeCodeConfig` accept an
      optional `templateContent`; omitting it reproduces prior behavior
      exactly (verified by existing generator tests passing unmodified).
- [x] `generateWorkspaceAdapters` generates real, non-empty
      `.opencode/AGENTS.md`, `.claude/AGENTS.md`, and
      `.github/copilot-instructions.md` on a fresh repo (no
      `axiom.spec/templates/*` present), with no `template-missing` /
      template-not-found warning for these three adapters.
- [x] `apps/cli/tests/workspace-adapters.test.ts`,
      `workspace-setup.test.ts`, `workspace-mcp.test.ts` updated to match
      (files now written; `template-missing` warnings gone for these
      adapters).
- [x] `npm run build` (tsc -b) clean from `Axiom/`.
- [x] Focused vitest run (`apps/cli`, `packages/adapters`,
      `packages/document-bootstrap`, `packages/installer`,
      `packages/user-workspace`) green (new + existing), except the known
      pre-existing unrelated failure in
      `packages/skills/tests/catalog.test.ts`.

## Open questions

None blocking — design decisions were supplied closed in the increment
prompt (bundle as TS constants, optional inline-template param, backward
compatible). Narrow implementation choices are recorded under
Assumptions.

## Assumptions

- The `templateContent` parameter takes precedence over `templatePath`
  when both happen to be provided (inline content wins) — no caller does
  this today, but the precedence order avoids ambiguity in the generator
  contract.
- `generateOpencodeConfig`/`generateClaudeCodeConfig`'s `readTemplate`
  disk-read helper is left untouched; the new parameter is handled as an
  early branch in the generator (skip `readTemplate` entirely when
  `templateContent` is provided) rather than changing `readTemplate`'s
  signature, to keep that helper's existing unit tests valid unmodified.
- `writeCopilotInstructions` required no code change — it already accepts
  template content as a string; only the caller
  (`generateWorkspaceAdapters`) changes, to pass the bundled constant
  instead of attempting a filesystem read.

## Implementation notes

- New file `apps/cli/src/commands/workspace-adapter-templates.ts`:
  exports `AGENTS_MD_TEMPLATE` and `COPILOT_INSTRUCTIONS_TEMPLATE`,
  verbatim copies of `Axiom.Spec/templates/agents-md-template.md` and
  `Axiom.Spec/templates/copilot-instructions.template.md`.
- `packages/adapters/opencode/src/types.ts` and
  `packages/adapters/claude-code/src/types.ts`: added
  `templateContent?: string` to `OpencodeConfigArgs`/`ClaudeCodeConfigArgs`.
- `packages/adapters/opencode/src/generator.ts` and
  `packages/adapters/claude-code/src/generator.ts`: when
  `args.templateContent` is set, use it directly as `template`, skipping
  the `readTemplate(templatePath)` disk read (and thus never producing
  `template-missing` for that call).
- `apps/cli/src/commands/workspace-adapters.ts`: `dispatchAdapterGenerator`
  passes `templateContent: AGENTS_MD_TEMPLATE` to both
  `generateOpencodeConfig`/`generateClaudeCodeConfig` calls, and replaces
  the copilot-vscode branch's filesystem `readFileSync` of
  `copilot-instructions.template.md` with the bundled
  `COPILOT_INSTRUCTIONS_TEMPLATE` constant (no more try/catch around a
  disk read for that path — it can no longer fail with
  template-not-found).
- Tests updated: `workspace-adapters.test.ts` (opencode/claude-code/
  copilot-vscode scenarios rewritten to assert real file content, no
  warnings), `workspace-setup.test.ts` (Scenario a/b/c/e — `warnings`
  assertions relaxed/removed since `template-missing` no longer fires for
  these adapters given the tmp repos used in those fixtures only select
  opencode/claude-code), `workspace-mcp.test.ts` (same).

## Validation

From `C:\repos\Axiom Workspace\Axiom`:

- `npm run build` (tsc -b) — clean, no errors.
- `npx vitest run apps/cli packages/adapters packages/document-bootstrap packages/installer packages/user-workspace`:

  ```
  Test Files  74 passed (74)
       Tests  645 passed (645)
  ```

  0 failures. New tests: 2 in `packages/adapters/opencode/tests/
  generator.test.ts` and 3 in `packages/adapters/claude-code/tests/
  generator.test.ts` (`templateContent` provided vs. omitted vs. neither
  provided). `[orchestrator] FAIL: command=...` lines seen during the run
  are pre-existing log output from OTHER tests that intentionally
  exercise failure paths (sync/init/join/start negative cases),
  confirmed by the `74 passed (74)` / `645 passed (645)` summary.
- `npx vitest run packages/skills` (out of the mandated scope, run only
  to confirm the pre-existing failure class): 1 failed / 70 passed — the
  known pre-existing, unrelated failure in
  `packages/skills/tests/catalog.test.ts > "carga el catálogo real del
  repo"` (`expected 'absent' to be 'ok'`), untouched by this increment.

## Result

Bundled `agents-md-template.md` and `copilot-instructions.template.md`
verbatim as TS constants (`apps/cli/src/commands/workspace-adapter-
templates.ts`). Added an optional `templateContent?: string` to
`OpencodeConfigArgs`/`ClaudeCodeConfigArgs`
(`packages/adapters/opencode/src/types.ts`,
`packages/adapters/claude-code/src/types.ts`); both generators
(`generator.ts` in each package) now skip the `readTemplate` disk read
entirely when `templateContent` is provided, otherwise behaving exactly
as before. `generateWorkspaceAdapters`
(`apps/cli/src/commands/workspace-adapters.ts`) now passes
`templateContent: AGENTS_MD_TEMPLATE` to both `generateOpencodeConfig`
and `generateClaudeCodeConfig`, and passes `COPILOT_INSTRUCTIONS_TEMPLATE`
directly as `writeCopilotInstructions`'s `template` (removing the
filesystem `readFileSync` of `copilot-instructions.template.md` that
previously always failed on a fresh repo). All three adapters
(opencode/claude-code/copilot-vscode) now produce real, non-empty
instruction files with zero `template-missing` warnings on a
freshly-scaffolded workspace repo; `configure.ts`/`sync.ts`/`init.json`
and the single-repo path were not touched and remain byte-for-byte
backward compatible (confirmed by every pre-existing generator/CLI test
passing unmodified in behavior). Build and the mandated focused test
suite are both green.

## General spec integration

Integrated into the canonical spec in the round-2 cross-increment pass
(covering this increment and the three sibling round-2
`INC-20260705-workspace-*` increments). Files updated:

- **06_Integraciones_y_Capacidades.md** — added the "Plantillas de adapter
  bundleadas (contenido real de instrucciones)" subsection:
  `AGENTS_MD_TEMPLATE`/`COPILOT_INSTRUCTIONS_TEMPLATE` bundled as TS
  constants, the optional `templateContent` param on opencode/claude-code
  generators, and opencode/claude-code/copilot-vscode now producing real
  non-empty instruction files with no `template-missing` warning.
- **01_Requisitos_Funcionales.md** — folded into the RF-AXM-024
  multi-select-adapters sub-point (noting the templates now make
  opencode/claude-code/copilot-vscode produce real instruction files).
- **08_Glosario.md** — added "Plantillas de adapter bundleadas".

No `05`/`03` change was needed beyond the sibling adapters increment's
edits: this increment only makes the already-described multi-adapter
generation non-degraded (real content instead of `template-missing`), not
a new data shape or interface contract.
