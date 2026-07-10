# Increment: Autoskills-style stack detection + per-code-repo skill wizard phase (AB12)

Status: closed
Date: 2026-07-08

## Goal

Bring `C:\repos\autoskills`'s core idea (auto-detect a repo's tech stack and
suggest/install matching curated skills) into Axiom, offered as an OPT-IN
**final phase** of the TUI workspace wizard: after workspace setup completes,
for EACH freshly-created CODE repo, detect its stack and let the user pick
(one-by-one, per repo) which curated skills to install. Installed skills must
land in that repo's `skills-index/<role>.yaml` (`available` list) so the
repo's implementation agents pick them up. Also expose the same
detect+suggest+install engine as a standalone `axiom skills suggest` command,
usable independently of the wizard.

## Context

- `C:\repos\autoskills` (`packages/autoskills/{lib.ts,skills-map.ts}`):
  marker-based stack detection (`package.json` deps, `*.csproj`,
  `angular.json`, gradle layout, `pyproject.toml`/`requirements.txt`, file
  extensions) mapped to a large (100+) curated skill registry
  (`SKILLS_MAP`/`COMBO_SKILLS_MAP`). Axiom does NOT ingest that registry —
  only the DETECTION SHAPE (which marker implies which tech) is reused as
  inspiration; the skill catalog itself stays small and curated per Axiom's
  existing bundled-seed convention.
- `apps/cli/src/commands/workspace-rules.ts`'s `inferRepoLanguages(repoPath)`
  (AB8) already does minimal marker-based detection for 4 ids
  (`typescript`/`csharp`/`angular`/`python`) via `package.json`/
  `tsconfig.json`/`*.csproj`/`angular.json`/`pyproject.toml`/
  `requirements.txt`. This increment REUSES it as the base signal and adds
  package.json dependency-based framework detection on top (react/next/
  express) without duplicating the file-marker logic.
- `apps/cli/src/commands/workspace-skills.ts` (AB-prior,
  INC-20260705-workspace-code-repo-skills): `scaffoldCodeRepoSkills`
  writes a bundled seed catalog (`axiom.config/skills-catalog.yaml`),
  materializes `SKILL.md` per skill via `@axiom/skills`'s
  `loadSkillRegistry`/`applySkillSet`, and writes
  `axiom.config/skills-index/<roleId>.yaml`. This increment's install path
  EXTENDS that same catalog (appends stack skill entries, does not
  replace/duplicate the seed) and extends that same role-index's
  `available` list.
- `apps/cli/src/commands/tui.ts`'s `runNoProjectBootstrap`: runs ONE
  `runTuiDriver({ wizardSteps })` call with a STATIC step list built BEFORE
  the driver starts, then (on confirm) calls `runWorkspaceSetup`, prints a
  summary, and re-opens the operative TUI on the control repo. The
  `multi-select` `WizardStep` kind (`packages/tui/src/driver.ts`) is generic
  (options/labels/defaults supplied by the app layer).
- `@axiom/skills` (`packages/skills/src/{catalog,materialize,apply,role-index}.ts`):
  `loadSkillsCatalog`/`materializeSingleSkill`/`computeSkillBundleHash`
  (append-friendly, id-keyed catalog) and
  `loadSkillsRoleIndex`/`validateSkillsRoleIndex` (role-index with
  `mandatory`/`available`, `available[].tags`/`summary`).

## Scope

1. **`apps/cli/src/commands/workspace-autoskills.ts`** (new, app-layer,
   mirrors `workspace-rules.ts`/`workspace-skills.ts`'s placement/pattern):
   - `STACK_SKILL_MAP`: curated, bundled TS constant mapping a detected tech
     id to one or more suggested skill ids (`react`, `nextjs`, `angular`,
     `dotnet-api`, `python`, plus the 4 `inferRepoLanguages` ids folded in
     as skill suggestions where a dedicated stack skill exists).
   - `STACK_SKILL_SOURCES`: bundled Markdown body per NEW stack skill id
     (`react`, `nextjs`, `dotnet-api`, `angular`, `python` — 5 new curated
     skills), same "bundled as TS constants" convention as
     `workspace-skills.ts`'s `CANONICAL_SEED_SOURCES` (`tsc -b` does not
     copy non-TS assets to `dist`).
   - `detectStack(repoPath) -> string[]`: reuses `inferRepoLanguages` for
     the file-marker signals (typescript/csharp/angular/python) and adds a
     `package.json`-dependency read for `react`/`next`/`express` (best-effort
     JSON parse, never throws). Read-only, local, synchronous.
   - `suggestSkillsForRepo(repoPath) -> { tech: string[]; skillIds: string[] }`:
     composes `detectStack` + `STACK_SKILL_MAP`, deduped, stable order.
   - `installSuggestedSkills(args) -> { filesCreated, warnings }`: appends
     the chosen stack skill ids' bundled sources + catalog entries into the
     repo's EXISTING `axiom.config/skills-catalog.yaml` (no-clobber: skips
     an id already present in the catalog), materializes the chosen ids via
     `loadSkillRegistry`/`applySkillSet` (same engine as
     `scaffoldCodeRepoSkills`), and appends them to
     `axiom.config/skills-index/<roleId>.yaml`'s `available` list (creates
     the role-index if absent; no-clobber on ids already listed). Never
     throws; a failure degrades to a warning and a partial result.
2. **Wizard final phase** (`apps/cli/src/commands/tui.ts`'s
   `runNoProjectBootstrap`): after a successful `runWorkspaceSetup` call
   (existing summary-print block), for each freshly-created `kind: 'role'`
   repo in `setupResult.repos`, run a SECOND `runTuiDriver` call (same
   input/output streams) with ONE dynamically-built `multi-select`
   `WizardStep` (title names the repo/role; options = that repo's
   `suggestSkillsForRepo` result; default: none checked). Confirming
   installs the chosen ids into that repo via `installSuggestedSkills`;
   declining/skipping (empty selection, or cancel) is a no-op for that repo.
   Runs repo-by-repo, sequentially. Best-effort: any exception for one repo
   is caught, reported as a warning line, and does not affect the other
   repos or the already-completed workspace setup.
3. **Standalone CLI**: `apps/cli/src/commands/skills.ts` gains
   `axiom skills suggest [--repo <path>] [--apply]` — prints detected tech +
   suggested skill ids for `--repo` (default: cwd); with `--apply`, calls
   `installSuggestedSkills` for the FULL suggested set and reports what was
   written. Mirrors the existing `axiom skills <subcommand>` shape
   (`withProjectContext`-free, since `--repo` may target a repo that is not
   the current project — same reasoning as `workspace-skills.ts`'s
   repo-path-as-argument functions).
4. **Availability to implementation agents**: both the wizard path and the
   CLI `--apply` path converge on the SAME `installSuggestedSkills`, so
   installed skills always land in `axiom.config/skills-index/<roleId>.yaml`'s
   `available` array — the artifact AB4/AB8's per-repo role wiring and the
   canonical `AGENTS.md` guidance already point implementation agents at.

## Non-goals

- Does NOT ingest autoskills' full 100+ skill registry or its combo-skill
  logic — `STACK_SKILL_MAP` is a small, curated, bundled set (5 new stack
  skills + reuse of existing seed ids where relevant).
- Does NOT change `@axiom/tui`'s driver to understand a new step kind or a
  "nested wizard" concept — the per-repo loop is implemented as N sequential
  `runTuiDriver` calls from the app layer, each with a single dynamically
  built step, reusing the existing generic `multi-select` kind unchanged.
- Does NOT integrate into `Axiom.Spec/specs/00-08` (reserved for the
  orchestrator's final consolidation pass) and does not git-commit.
- Does NOT add new npm dependencies.
- Does NOT modify `scaffoldCodeRepoSkills`'s existing seed baseline
  behavior — this increment only ADDS an optional, later, additive
  extension path over the same catalog/role-index files.

## Acceptance criteria

- [x] `detectStack(repoPath)` returns the right tech ids for react/dotnet/
      angular/python fixture repos (marker-based, reusing
      `inferRepoLanguages` + `package.json` dependency detection).
- [x] `suggestSkillsForRepo(repoPath)` composes detection + `STACK_SKILL_MAP`
      into a deduped `{ tech, skillIds }`.
- [x] `installSuggestedSkills` appends chosen skills' bundled sources +
      catalog entries into an EXISTING `skills-catalog.yaml` (idempotent,
      no-clobber on already-present ids) and appends them into
      `skills-index/<roleId>.yaml`'s `available` list; `SKILL.md` gets
      materialized for each chosen id.
- [x] `axiom skills suggest [--repo] [--apply]` CLI works standalone.
- [x] Wizard: a scripted per-repo selection after workspace setup installs
      the chosen skills into that code repo (tui.test.ts extended).
- [x] Full suite stays green; final `--no-file-parallelism` run shows 0
      regressions vs. the batch baseline.

All satisfied — see Result/Validation for verbatim evidence.

## Open questions

None blocking — ambiguities resolved below (Assumptions).

## Assumptions

- **`STACK_SKILL_MAP` scope**: exactly 5 NEW curated stack skill ids
  (`react`, `nextjs`, `angular`, `dotnet-api`, `python`), each mapping 1:1
  to one detected tech id from `detectStack`. This is deliberately far
  smaller than autoskills' registry, per the brief's "curated (do NOT
  ingest autoskills' 100+ registry)" constraint. A repo that also matches
  `csharp`/`typescript` (already detected by `inferRepoLanguages`) gets
  offered the existing bundled RULES for those languages separately
  (AB8, unchanged) — this increment's suggestions are ADDITIVE skill
  offers layered on top, not a replacement for the rules layer.
- **`express`/`react`/`next` detection**: added as a thin `package.json`
  `dependencies`/`devDependencies` key-membership check (best-effort JSON
  parse, catches malformed `package.json` and treats it as "no deps
  detected" rather than throwing) — the same shape autoskills uses for
  its `packages` detect field, scoped to exactly the 3 ids this increment
  needs (`react`, `next` -> `nextjs` tech, `express` folded into the
  `react`/generic node suggestion set is NOT added — `express` has no
  bundled stack skill in this curated set, so it is detected for
  completeness but does not yet map to a suggestion; documented here, not
  silently dropped).
- **Per-repo wizard loop mechanics**: because `RunTuiArgs.wizardSteps` must
  be a static list resolved before `runTuiDriver` starts, and the set of
  code repos + their detected stacks is only known AFTER
  `runWorkspaceSetup` returns, the phase is implemented as N SEQUENTIAL
  `runTuiDriver({ initialScreen: 'setup', wizardSteps: [oneMultiSelectStep] })`
  calls (one per freshly-created role repo), reusing the exact same
  input/output streams. This is architecturally honest given the current
  driver design (documented in the brief as an acceptable outcome) rather
  than inventing a new "nested/dynamic wizard" driver concept. Each of
  these mini-wizards shows a single `multi-select` + the existing generic
  confirmation step; confirming (even with zero items checked) or
  cancelling both move on to the next repo without affecting the others.
- **Repos offered the phase**: only `kind: 'role'` repos that were FRESHLY
  CREATED in this same `runWorkspaceSetup` call (`setupResult.repos[].created
  === true` AND the corresponding `spec.repos[]` entry has
  `kind: 'role'`) — mirrors the exact same gate `scaffoldCodeRepoSkills`
  already uses in `workspace-setup.ts`'s step 7b, for the same reason
  (never touch a pre-existing repo's skills without an explicit, separate
  invocation — the standalone CLI is that explicit invocation path for
  pre-existing repos).
- **Opt-out UX**: the phase always runs (per freshly-created role repo) when
  `detectStack` returns at least one suggestion; if `suggestSkillsForRepo`
  returns zero skill ids for a repo (unrecognized/empty stack), that repo's
  mini-wizard step is skipped entirely (no empty prompt shown). The user can
  always decline per-repo by confirming with nothing checked.

## Implementation notes

**Files created:**
- `Axiom/apps/cli/src/commands/workspace-autoskills.ts` — `detectStack`,
  `suggestSkillsForRepo`, `installSuggestedSkills`, `STACK_SKILL_MAP`,
  bundled `STACK_SKILL_SOURCES`.
- `Axiom/apps/cli/tests/workspace-autoskills.test.ts`.

**Files edited:**
- `Axiom/apps/cli/src/commands/tui.ts` — new exported
  `runAutoskillsWizardPhase` (per-code-repo skill-suggestion phase),
  wired into `runNoProjectBootstrap` right after a successful
  `runWorkspaceSetup` call and before re-opening the operative TUI.
  Exported (not module-private) specifically so tests can drive it
  directly with a synthetic `spec`/`setupResult` instead of only through
  the full interactive end-to-end wizard sequence (see Implementation
  notes addendum for why: a freshly-created role repo never has a stack
  marker by default, so the real end-to-end path always skips the
  prompt).
- `Axiom/apps/cli/src/commands/skills.ts` — new `runSkillsSuggest` +
  `axiom skills suggest [--repo] [--apply]` subcommand.
- `Axiom/apps/cli/tests/tui.test.ts` — new "Scenario 8" describe block (4
  tests) exercising `runAutoskillsWizardPhase` directly: install-on-confirm,
  decline-with-nothing-checked, no-prompt-when-nothing-suggested, and
  skip-preexisting-repo.
- `Axiom/apps/cli/tests/skills.test.ts` — new "Scenario 6/7" (3 tests):
  `suggest` read-only + `suggest --apply` install-and-report.

## Implementation notes (addendum)

- **Bug found + fixed while implementing**: `renderRoleIndexYaml`'s
  `mandatory:`/`available:` keys, when the array is empty, must be
  rendered as `mandatory: []`/`available: []` (flow-style empty array).
  Emitting just `mandatory:` with no items underneath parses via `js-yaml`
  as `null`, not `[]`, which fails `validateSkillsRoleIndex`'s
  `Array.isArray` check. This module's OWN copy of the helper (in
  `workspace-autoskills.ts`, independently re-declared from
  `workspace-skills.ts`'s identical-in-spirit helper, same convention as
  that module already uses for its own trivial re-declarations) needed
  this fix because `installSuggestedSkills` can create a role-index from
  scratch with an empty `mandatory` (it never populates `mandatory` — that
  is `scaffoldCodeRepoSkills`'s job). `workspace-skills.ts`'s
  `renderRoleIndexYaml` has the SAME latent bug (never triggered there
  because its `buildRoleIndex` always populates a non-empty `mandatory`) —
  left unchanged since it is out of this increment's scope (not touched,
  no observed failure), but worth flagging for the general-spec pass /
  a future increment that touches that file.

## Validation

`npm run build` (`tsc -b`, from `Axiom/`):
```
> axiom-product@0.1.0 build
> tsc -b
```
(clean, no output, exit 0.)

`npx vitest run apps/cli packages/skills packages/tui`:
```
 Test Files  78 passed (78)
      Tests  828 passed (828)
```

`npx vitest run --no-file-parallelism` (full suite, serial):
```
 Test Files  198 passed (198)
      Tests  2119 passed (2119)
```
0 regressions vs. the batch baseline (2088/2088 going into this
increment); delta is `+31` new tests, all new and all passing:
`apps/cli/tests/workspace-autoskills.test.ts` (new file, 24 tests),
`+4` new scenarios in `apps/cli/tests/tui.test.ts` (26 -> 30), `+3` new
scenarios in `apps/cli/tests/skills.test.ts` (5 -> 8).

## Result

Autoskills-style stack detection now exists natively in Axiom, curated and
100% local. `apps/cli/src/commands/workspace-autoskills.ts` adds
`detectStack(repoPath)` (reuses AB8's `inferRepoLanguages` for file-marker
signals — `typescript`/`csharp`/`angular`/`python` — plus a best-effort
`package.json` dependency read for `react`/`nextjs`/`express`),
`suggestSkillsForRepo(repoPath)` (composes detection against a curated,
bundled `STACK_SKILL_MAP` covering exactly 5 new stack skills: `react`,
`nextjs`, `angular`, `dotnet-api`, `python`), and
`installSuggestedSkills(args)` (appends the chosen skills' bundled sources
+ catalog entries into the repo's EXISTING `axiom.config/skills-catalog.yaml`
— no-clobber by id — materializes `SKILL.md` via the same
`loadSkillRegistry`/`applySkillSet` engine `scaffoldCodeRepoSkills` already
uses, and appends the chosen ids into
`axiom.config/skills-index/<roleId>.yaml`'s `available` list, creating that
file if absent).

The TUI workspace wizard's final phase
(`apps/cli/src/commands/tui.ts`'s `runAutoskillsWizardPhase`, wired into
`runNoProjectBootstrap` right after a successful `runWorkspaceSetup` call)
offers this ONE-BY-ONE, per freshly-created code repo: for each `kind:
'role'` repo with `created === true` in that same setup run, it detects the
stack and — only if at least one skill is suggested (never an empty
prompt) — runs a SEPARATE `runTuiDriver` call with a single dynamically
built `multi-select` step naming that repo's suggested skills. The user
picks (or declines with nothing checked); confirming installs the choice
into that repo via `installSuggestedSkills`. This is implemented as N
sequential `runTuiDriver` invocations (one per fresh role repo) rather than
one combined multi-step wizard, because `RunTuiArgs.wizardSteps` must be a
static list resolved BEFORE the driver starts, while the set of code repos
and their detected stacks is only known AFTER `runWorkspaceSetup` returns —
documented as the honest, non-speculative resolution of that structural
constraint (see Assumptions) rather than inventing a new "nested/dynamic
wizard" concept in the generic `@axiom/tui` driver.

The same engine is exposed standalone via `axiom skills suggest [--repo
<path>] [--apply]` (`apps/cli/src/commands/skills.ts`'s
`runSkillsSuggest`), usable against ANY repo path (not just the active
project), mirroring the existing `axiom skills <subcommand>` shape. Without
`--apply` it only prints detected tech + suggested skill ids (read-only);
with `--apply` it installs the full suggested set via the same
`installSuggestedSkills` used by the wizard phase.

Both paths converge on `axiom.config/skills-index/<roleId>.yaml`'s
`available` array — the same artifact AB4/AB8's per-repo role wiring and
the canonical `AGENTS.md` guidance already point that repo's
implementation agents at — so installed skills are immediately visible to
them with no additional wiring.

A real bug was found and fixed while building this
(`renderRoleIndexYaml`'s empty-array rendering — see Implementation notes
addendum); it is currently latent-but-dormant in
`workspace-skills.ts`'s identical helper and flagged for the general-spec
pass / a future touch of that file.

## General spec integration

Integrated into `Axiom.Spec/specs/00-08` by the batch's final
consolidation pass (autopilot orchestrator). Landed in:

- **06_Integraciones_y_Capacidades.md** — PRIMARY. New subsection
  "Detección de stack + sugerencia de skills (autoskills)":
  `detectStack`/`suggestSkillsForRepo`/`installSuggestedSkills`, the
  curated `STACK_SKILL_MAP` (5 stack skills), and convergence on
  `skills-index/<roleId>.yaml#available`; noted as coexisting with the
  skills baseline and the rules layer.
- **05_Interfaces_Operativas.md** — the wizard's autoskills final phase
  (per freshly-created code repo) and the standalone `axiom skills
  suggest [--repo] [--apply]` command.
- **08_Glosario.md** — new term: autoskills.

The latent `renderRoleIndexYaml` empty-array-renders-as-null bug (still
dormant in `workspace-skills.ts`) is an implementation follow-up flagged
here, not stable spec knowledge, so it stays in this README (not folded
into 00-08).
