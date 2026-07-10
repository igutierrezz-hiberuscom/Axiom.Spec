# Increment: Language-scoped rules layer (AB8)

Status: closed
Date: 2026-07-08

## Goal

Add a curated, LOCAL, bundled, LANGUAGE-SCOPED coding-standard "rules" layer
to Axiom — inspired by (not ported from) `C:\repos\ECC`'s `rules/common/`,
`rules/typescript/`, etc. — so a scaffolded workspace gets a small set of
concise, human-readable rule docs at a canonical per-project location, wired
best-effort into `runWorkspaceSetup`, referenced from the canonical
`AGENTS.md`, and (where a real native location exists) projected into an
adapter-native rules file.

## Context

- `C:\repos\ECC` distributes ~20 language-scoped rule directories
  (`rules/common`, `rules/typescript`, `rules/python`, `rules/golang`,
  `rules/csharp`, ...), each with several files (`coding-style.md`,
  `testing.md`, `security.md`, ...), copied to the agent's rules dir on
  install. Axiom has no equivalent layer today. The brief is explicit: do
  NOT port ECC's whole library — bundle a small, curated catalog instead
  (`common` + a handful of languages relevant to Axiom itself and typical
  client stacks).
- `packages/skills/src/materialize.ts` establishes the "bundle content as TS
  constants, write atomically (tmp + rename)" pattern already used for
  skills (`tsc -b` does not copy non-TS asset files to `dist`, so a runtime
  `readFileSync` against a template directory would break in a built
  package).
- `apps/cli/src/commands/workspace-skills.ts` (`scaffoldSddSkills`/
  `scaffoldCodeRepoSkills`) and `apps/cli/src/commands/workspace-spec-base.ts`
  (`scaffoldSpecRepoBase`) are the two existing "bundle TS constants + write
  best-effort into a newly-scaffolded repo" precedents this increment
  mirrors directly. `scaffoldSpecRepoBase` in particular already establishes
  the exact no-clobber pattern needed here: check `fs.existsSync` per file
  BEFORE calling `@axiom/document-bootstrap`'s `writeGuardedFile`, and push a
  warning instead of overwriting.
- `apps/cli/src/commands/workspace-setup.ts`'s `runWorkspaceSetup` is the
  best-effort multi-repo scaffolding engine; it already has a `created`-gated
  step pattern (skills only scaffold into a repo that was newly created in
  step 1) that this increment's wiring step follows for the CONTROL repo,
  but deliberately does NOT gate on `created` for rules in code repos (see
  Assumptions — rules are cheap, additive, per-file no-clobber, and useful
  even on a pre-existing repo being re-configured).
- `@axiom/document-bootstrap`'s `writeCanonicalAgentsMd`/
  `renderCanonicalAgentsMd` render the canonical, repo-root `AGENTS.md`
  (`AXIOM:GENERATED`/`TEAM:CUSTOM` block preservation already handled there).
- `packages/adapters/cursor/src/generator.ts` does not write anything under
  `.cursor/rules/*` today — confirmed by reading the file — so adding a
  native `.cursor/rules/axiom-common.mdc` projection there is a genuinely
  new, additive surface with no existing behavior to preserve/collide with.
- `apps/cli/src/commands/init.ts` exports `atomicWriteFile` (used across
  `workspace-setup.ts`) but this increment reuses
  `@axiom/document-bootstrap`'s `writeGuardedFile` instead (already the
  established shared atomic-write-with-path-guard primitive; avoids a
  needless second implementation).

## Scope

- New module `apps/cli/src/commands/workspace-rules.ts` (app-layer, mirrors
  `workspace-skills.ts`/`workspace-spec-base.ts` — NOT a new package; see
  Assumptions for why a package was rejected). Bundles a curated rule
  catalog as TS string constants:
  - `common` — language-agnostic Axiom/SDD engineering guidelines
    (`Result<T,E>` over throwing, atomic writes, no speculative
    architecture, test discipline, spec-first workflow).
  - `typescript`, `python`, `csharp`, `angular` — one concise rule doc each.
- `scaffoldRules({ repoPath, languages })`: writes `axiom.config/rules/
  common.md` (always) plus `axiom.config/rules/<language>.md` for each
  requested language, best-effort, per-file no-clobber (never overwrites an
  existing file — pushes a warning instead). Returns `{ filesCreated,
  warnings, scopesWritten, scopesPresent }` — `scopesPresent` (union of
  newly-written + already-present scopes) is what the caller must use to
  populate `AGENTS.md`'s rule-scope listing, so the section does not
  disappear on a re-run where nothing new gets written (see Validation's
  "real bug found and fixed" note). A companion pure helper,
  `scopesPresentOnDisk(repoPath)`, lets a caller ask what is present
  without invoking `scaffoldRules` at all (used for the control repo when
  it is not newly-created).
- `inferRepoLanguages(repoPath)`: minimal, marker-file-based language
  inference (`package.json`/`tsconfig.json` → `typescript`; `*.csproj` →
  `csharp`; `angular.json` → `angular`; `pyproject.toml`/
  `requirements.txt` → `python`); returns `[]` when no marker matches
  (caller still gets `common`).
- Wiring into `runWorkspaceSetup`:
  - control repo: `common` only, gated on the control repo being newly
    created (mirrors the existing skills/spec-base `created` gate).
  - each code (`kind: 'role'`) repo: `common` + inferred languages for that
    repo's path (best-effort, never fails setup); NOT gated on `created`
    (rules are safe/no-clobber to add to a pre-existing code repo being
    wired into a workspace).
- `AGENTS.md`: new short "Coding rules" section in
  `renderCanonicalAgentsMd`, listing `axiom.config/rules/<scope>.md` for
  each scope actually present on disk at render time (best-effort read,
  never throws) — mirrors the existing conditional "Code intelligence
  tools" section pattern.
- Native adapter projection: `packages/adapters/cursor/src/generator.ts`
  gains a best-effort projection of the `common` rule doc (only) into
  `.cursor/rules/axiom-common.mdc` (cursor's real, documented native rules
  location), with the same no-clobber-if-user-edited guard the rest of the
  cursor generator already uses for its other generated files (byte-compare
  against the last-generated content when tracked, else "already exists"
  skip).
- No new CLI surface (`axiom rules list`/`apply`) — see Assumptions.

## Non-goals

- Not porting ECC's ~20-language, dozens-of-files rule library. Four
  languages + `common`, one concise file each.
- No new npm dependencies.
- No new `packages/rules` package (rejected — see Assumptions).
- No `axiom rules` CLI command (rejected for this increment — see
  Assumptions).
- No git commits, ever.
- No integration into `Axiom.Spec/specs/00-08` (reserved for the
  autopilot's final consolidation pass across the batch).
- No changes to `@axiom/tui` (stays generic; does not know about rules).
- No projection into every adapter's native format — only `.cursor/rules`,
  because it is the only adapter with a confirmed, real, currently-unused
  native rules location. Other adapters (opencode, claude-code,
  github-copilot, vscode, litellm) are not touched.

## Acceptance criteria

- [x] `scaffoldRules` writes `axiom.config/rules/common.md` plus each
      requested language file, non-empty, sourced from the bundled TS
      constants.
- [x] `scaffoldRules` never overwrites a pre-existing rule file (no-clobber)
      and reports a warning naming the file it skipped.
- [x] `inferRepoLanguages` returns the correct scope set for fixture repos
      carrying `package.json`/`tsconfig.json` (typescript), a `*.csproj`
      (csharp), `angular.json` (angular), and `pyproject.toml`/
      `requirements.txt` (python) marker files, and `[]` for a repo with no
      recognized marker.
- [x] `runWorkspaceSetup` produces `axiom.config/rules/common.md` in the
      control repo (gated on `created`) and `common` + inferred language
      rule files in each code repo, best-effort (never throws, never fails
      setup on a write error).
- [x] Canonical `AGENTS.md` gains a short "Coding rules" section pointing at
      the rule files actually present.
- [x] `.cursor/rules/axiom-common.mdc` is best-effort projected when the
      cursor adapter is selected, without clobbering a user-edited file.
- [x] `npm run build` (`tsc -b`) is clean.
- [x] `npx vitest run apps/cli packages/document-bootstrap
      packages/adapters/cursor` passes; full suite
      (`npx vitest run --no-file-parallelism`) shows 0 regressions vs. the
      `1995/1995` baseline.

## Open questions

None blocking. Ambiguities resolved under Assumptions.

## Assumptions

- **App-layer module, not a new package**: the brief offered a choice.
  `workspace-skills.ts`/`workspace-spec-base.ts` (the two closest existing
  precedents for "bundle curated TS-constant content + write best-effort
  into a scaffolded repo") both live at `apps/cli/src/commands/`, are
  consumed only by `workspace-setup.ts`, and are NOT re-exported through
  `@axiom/cli-commands`'s barrel (confirmed by reading
  `packages/cli-commands/src/index.ts` — neither file appears there) nor
  through the TUI. A new `packages/rules` package would only be justified if
  something outside `apps/cli` needed to import the catalog directly; no
  such consumer exists or was requested. Mirroring the established sibling
  pattern (same directory, same shape, same tsconfig — `apps/cli`'s own
  `tsconfig.json` compiles `workspace-rules.ts` directly, same as
  `workspace-skills.ts`/`workspace-spec-base.ts`, confirmed NOT present in
  either `tsconfig.json`'s `exclude` list) keeps the change small and
  avoids a speculative package boundary.
- **No `axiom rules` CLI command**: the brief explicitly allows
  "scaffolding-on-setup is sufficient — note the decision" as a fallback.
  `runWorkspaceSetup` itself has no standalone CLI entry point (it is
  invoked by the wizard/TUI, not a directly-registered `commander` command
  today — confirmed: no `registerWorkspace*` in `apps/cli/src/index.ts`), so
  adding `axiom rules list/apply` would require inventing a new top-level
  command family disconnected from how rules actually get scaffolded today
  (via workspace setup), for a curated catalog with only 5 scopes. Deferred
  as a future consideration if/when a real need for standalone
  list/re-apply surfaces (e.g. alongside a future `axiom skills` -like
  drift/refresh workflow for rules).
- **Code repos are NOT gated on `created`**: unlike skills/spec-base (which
  only scaffold into a brand-new repo, since re-running them against an
  existing repo would be redundant/wasteful), rules scaffolding is cheap,
  purely additive, and per-file no-clobber. A repo that already exists
  (e.g. an existing backend already wired into a new Axiom workspace) still
  benefits from getting `axiom.config/rules/*` the first time
  `runWorkspaceSetup` runs against it. This is a deliberate, narrow
  deviation from the skills/spec-base gating precedent, justified by rules
  having no destructive-overwrite risk (each file is independently
  no-clobber) where skills/spec-base's materialization engine does not make
  the same per-file guarantee at the same granularity.
- **Language inference is marker-file-only, best-effort, non-exhaustive**:
  exactly the four checks the brief names
  (`package.json`/`tsconfig.json`->typescript, `*.csproj`->csharp,
  `angular.json`->angular, `pyproject.toml`/`requirements.txt`->python).
  A repo matching none of these gets `common` only — never a hard failure,
  never a guess.
- **`angular` detection does not exclude `typescript`**: an Angular repo
  also has a `package.json`/`tsconfig.json`, so it legitimately gets BOTH
  `typescript.md` and `angular.md` (Angular rules supplement, not replace,
  general TypeScript rules) — the four checks are independent, not
  mutually exclusive branches.
- **`.cursor/rules/axiom-common.mdc` only projects `common`**, not every
  scaffolded language scope: cursor's `.mdc` rule files are typically
  scoped per-concern by the user; projecting one curated, always-relevant
  file (the language-agnostic baseline) avoids guessing which of a
  multi-language workspace's per-language rule files cursor's single active
  project should receive. `axiom.config/rules/` remains the canonical
  source for the full scope set regardless of adapter.
- **`AGENTS.md`'s "Coding rules" section reads the filesystem at render
  time** (not from `WorkspaceSetupSpec` state) — consistent with
  `renderCanonicalAgentsMd` being otherwise a pure function; this one
  section is rendered by `writeCanonicalAgentsMd`'s caller
  (`workspace-setup.ts`), which lists whichever `axiom.config/rules/*.md`
  files were actually just scaffolded for that repo, so the pure renderer
  itself stays synchronous/pure (an explicit `ruleScopes?: readonly
  string[]` field on `CanonicalAgentsMdIdentity`, populated by the caller —
  same shape as the existing `enabledCodeIntelProviders` field).

## Implementation notes

Files created:
- `Axiom/apps/cli/src/commands/workspace-rules.ts` — bundled rule catalog
  (`common`, `typescript`, `python`, `csharp`, `angular`), `scaffoldRules`
  (returns `scopesWritten` + `scopesPresent`), `inferRepoLanguages`,
  `scopesPresentOnDisk` (pure disk-read helper), `RULE_SCOPES`.
- `Axiom/apps/cli/tests/workspace-rules.test.ts` — catalog content,
  no-clobber, language inference, `scopesPresent`/`scopesPresentOnDisk`.

Files edited:
- `Axiom/apps/cli/src/commands/workspace-setup.ts` — new best-effort step:
  scaffolds `common` into the control repo (gated on `created`, same
  pattern as skills) and `common` + inferred languages into every code
  repo (not gated on `created`); threads the resulting rule scopes into
  `writeCanonicalAgentsMd`'s identity per repo.
- `Axiom/packages/document-bootstrap/src/canonical-agents-md.ts` — new
  optional `CanonicalAgentsMdIdentity.ruleScopes` field and a "Coding
  rules" section, emitted only when non-empty.
- `Axiom/packages/document-bootstrap/tests/canonical-agents-md.test.ts` —
  new "Coding rules" section scenarios.
- `Axiom/packages/adapters/cursor/src/generator.ts` — best-effort
  `.cursor/rules/axiom-common.mdc` projection of the bundled `common` rule
  doc, no-clobber against user edits (same generated-content-tracking guard
  already used for the adapter's other generated files).
- `Axiom/packages/adapters/cursor/tests/generator.test.ts` — new
  `.cursor/rules/axiom-common.mdc` scenarios (Scenario 4) + Scenario 1's
  `writtenFiles` assertion updated to include the new third entry.
- `Axiom/apps/cli/tests/sync.test.ts` — Scenario 6's `cursor` case updated
  (`expectedFiles`/`expectedCount`) to account for the new
  `.cursor/rules/axiom-common.mdc` output.
- `Axiom/apps/cli/tests/workspace-setup.test.ts` — new Scenario (l) (8
  tests, including a dedicated regression test for the `scopesPresent` bug
  described below); Scenario (c)'s zero-warnings assertion updated to
  expect exactly one no-clobber warning on the idempotent second run (see
  Validation for why).

## Validation

- `npm run build` (`tsc -b`, from `Axiom/`): clean, no errors.
- `npx vitest run apps/cli packages/document-bootstrap packages/adapters/cursor`:
  ```
  Test Files  65 passed (65)
       Tests  599 passed (599)
  ```
- `npx vitest run --no-file-parallelism` (full suite):
  ```
  Test Files  193 passed (193)
       Tests  2030 passed (2030)
  ```
  Baseline (AB7 closing state, re-confirmed before starting this increment)
  was `1995/1995`; delta is `+35` new tests, `0` regressions
  (`workspace-rules.test.ts`: new file, 20 tests; `workspace-setup.test.ts`
  Scenario (l): +8; `canonical-agents-md.test.ts` "Coding rules" scenarios:
  +4; `packages/adapters/cursor/tests/generator.test.ts` Scenario 4: +3).
  Two pre-existing assertions were updated (not weakened) to reflect new,
  intentional behavior: `workspace-setup.test.ts` Scenario (c) (idempotent
  re-run) now expects exactly one no-clobber warning for the `backend`
  repo's `axiom.config/rules/common.md` on the second run (rules
  scaffolding for code repos is deliberately NOT gated on `created` — see
  Assumptions); `packages/adapters/cursor/tests/generator.test.ts` Scenario
  1 and `apps/cli/tests/sync.test.ts` Scenario 6's `cursor` case now expect
  the new `.cursor/rules/axiom-common.mdc` in `writtenFiles`/
  `generatedFilesCount`.

**Real bug found and fixed during test-authoring**: the first
implementation populated `CanonicalAgentsMdIdentity.ruleScopes` from
`scaffoldRules`'s `scopesWritten` (files written in THIS invocation only).
On any re-run where a rule doc already existed (no-clobber skip — the
common case for a control repo no longer gated `created`, or any code repo
after its first run), `scopesWritten` was `[]`, so the "Coding rules"
section silently disappeared from AGENTS.md on every re-run after the
first (step 1's `writeOneRepo` always regenerates the `AXIOM:GENERATED`
block from scratch without `ruleScopes`). Fixed by adding
`scaffoldRules`'s `scopesPresent` (union of newly-written + already-present
scopes) and a new pure `scopesPresentOnDisk(repoPath)` helper (used when
the control repo isn't newly-created, so `scaffoldRules` isn't even
invoked for it), and threading THAT into the AGENTS.md re-render instead.
Caught by a dedicated regression test (`workspace-setup.test.ts` Scenario
(l): "la sección 'Coding rules' persiste... a través de un re-run") that
runs `runWorkspaceSetup` twice and asserts the section survives on both the
control and the code repo's `AGENTS.md`.

## Result

Axiom now has a curated, LOCAL, bundled, language-scoped rules layer:
`common` (Axiom/SDD engineering guidelines) plus `typescript`/`python`/
`csharp`/`angular`, each a single concise Markdown file bundled as a TS
string constant in `apps/cli/src/commands/workspace-rules.ts`.
`scaffoldRules` writes them to the canonical `axiom.config/rules/<scope>.md`
location, best-effort and per-file no-clobber. `runWorkspaceSetup` wires
`common` into the control repo (once, on creation) and `common` + inferred
languages into every code repo (marker-file inference: `package.json`/
`tsconfig.json`->typescript, `*.csproj`->csharp, `angular.json`->angular,
`pyproject.toml`/`requirements.txt`->python), never failing setup. The
canonical `AGENTS.md` gained a short, conditional "Coding rules" section
pointing agents at the scaffolded files, and the cursor adapter gained a
best-effort, no-clobber `.cursor/rules/axiom-common.mdc` native projection
of the `common` doc — the only adapter with a confirmed, currently-unused
native rules location. No new CLI surface and no new package were added;
both were deliberate, documented decisions (see Assumptions).

## General spec integration

Integrated into `Axiom.Spec/specs/00-08` by the batch's final
consolidation pass (autopilot orchestrator). Landed in:

- **06_Integraciones_y_Capacidades.md** — PRIMARY. New subsection "Capa
  de reglas de código language-scoped (`axiom.config/rules/`)": bundled
  curated catalog (`common` + typescript/python/csharp/angular),
  `scaffoldRules`/`inferRepoLanguages`, `runWorkspaceSetup` wiring,
  AGENTS.md "Coding rules" section, and the `.cursor/rules/axiom-common.mdc`
  native projection.
- **03_Modelo_Operativo_y_Datos.md** — rules artifacts
  (`axiom.config/rules/<scope>.md`, `CanonicalAgentsMdIdentity.ruleScopes`)
  in the INC-20260708 data subsection.
- **08_Glosario.md** — new term: rules-layer (`axiom.config/rules/`).

The "`axiom rules` standalone CLI as a deferred future consideration"
note stays in this README (it is a follow-up, not stable end-state
knowledge).
