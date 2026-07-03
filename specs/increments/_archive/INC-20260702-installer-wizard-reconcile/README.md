# Increment: Reconcile project installation wizard (installer + install-profiles + init/join/repo/configure commands)

Status: pending
Date: 2026-07-02

## Goal

Audit the current Axiom project-installation flow (`@axiom/installer`,
`@axiom/install-profiles`, and the CLI commands `init`, `join`, `configure`,
`repo attach`) against the target installation questionnaire described in
the two external decision documents (source doc section 13.2 + addendum),
in light of the now-landed INC-01 (registry v2, `repos: { roleKey: {role,
path} }`) and INC-02 (`RepoRegistered`-style notifications). Produce a
concrete, evidence-grounded gap analysis and a brief for the next role.
This increment is **audit-only**: no code changes in `Axiom/`.

## Context

This is INC-03 of the parent roadmap
(`Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/README.md`),
under Phase B ("Instalación de proyecto"). INC-01 (registry + manifest
schema v2) and INC-02 (structural command foundation / event model) are
both closed. Their landed state (confirmed by direct inspection during
this audit, not assumed from the roadmap's original text):

- `@axiom/user-workspace` has `addProjectV2`/`removeProjectV2`/
  `getProjectV2`/`listProjectsV2`/`useProjectV2`/`findByRepoPathV2`
  reading/writing `~/.axiom/projects.yml` (registry v2, YAML), with a
  `repos: { roleKey: { role, path } }` map per project entry
  (`packages/user-workspace/src/registry-types.ts`). v1
  `registry.json`/`ProjectEntry` functions remain for backward compat.
- `@axiom/config-validation` has `AxiomYamlSchemaV2` (Zod,
  `schemaVersion: 2`, `projectId`/`name`/`repoId`/`role`/`paths`/
  `indexes`) alongside the unchanged v1 `AxiomYamlSchema`
  (`project.name`/`project.mode`/`scopes`), and a version-dispatching
  `validateAxiomYamlContent`.
- `@axiom/project-resolution`'s `resolveProject` (`resolver.ts`) is
  **already version-aware**: it branches on `validateAxiomYamlContent`'s
  `version` field and calls `resolveFromV1` or `resolveFromV2`, both
  normalized to the same external `ProjectResolution` shape. This
  directly contradicts a comment left in `init.ts` (see Finding 5 below).
- `apps/cli/src/commands/repo.ts`'s `runRepoAttach` already accepts an
  explicit `--role` flag (default `'sdd'`) and calls `addProjectV2` to
  attach an additional repo (any role: `sdd`/`spec`/`code`/etc.) to an
  existing or new project id in the v2 registry.
- `init.ts`, `join.ts`, and `repo.ts` all call `notifyRepoRegistered`
  (`_shared.ts`) after a successful `addProjectV2`, writing a
  `[axiom] event=RepoRegistered ...` line to stderr — this is INC-02's
  event model, implemented as a single traceable log line, not a bus.

## Scope

- Read `@axiom/installer` (`installer.ts`, `registry.ts` — the
  generated-files map, NOT the project registry — `profiles-loader.ts`,
  `persist.ts`, `types.ts`, `index.ts`) and its tests in full.
- Read `@axiom/install-profiles` (`composer.ts`, `constants.ts`,
  `types.ts`, `role-aliases.ts`, `overhead.ts`, `schemas.ts`, `index.ts`)
  and its tests in full.
- Read `apps/cli/src/commands/init.ts`, `join.ts`, `configure.ts`,
  `repo.ts`, `_shared.ts` in full, in their current post-INC-01/INC-02
  state.
- Produce a 6-dimension gap analysis (identity, repos, roles, adapters,
  tools/MCPs, bootstrap) comparing the current flow to source doc 13.2 +
  addendum.
- Make a concrete, evidence-based recommendation on whether `init.ts`'s
  deferred `schemaVersion: 2` re-enablement belongs in this increment's
  scope or should stay a separate task.
- Produce a concrete brief for the next role in the roadmap's planned
  sequence (docs-skills-writer), including whether `init` already emits
  an `AGENTS.md` (it does not — see Findings).

## Non-goals

- No code implementation. This increment produces findings and a brief
  only.
- No decision on Q1/Q2/Q5 (already resolved by INC-01's landed state,
  confirmed here by re-reading the actual code, not re-litigated).
- No redesign of `@axiom/install-profiles`' composition algorithm
  (`resolveInstallProfile`'s 10 internal steps) — that machinery is
  reusable as-is per the roadmap's own diff hypothesis, confirmed here.
- No decision on whether `single-repo` mode (`init.ts`'s
  `self-hosted`/`installed-multi-repo` layout inference) is deprecated —
  out of scope for this audit.
- No AGENTS.md template authoring — that is the next role's job
  (docs-skills-writer), scoped by this increment's brief, not executed
  here.

## Acceptance criteria

- [x] `Axiom/packages/installer/src/*` and its tests read in full.
- [x] `Axiom/packages/install-profiles/src/*` and its tests read in full.
- [x] `Axiom/apps/cli/src/commands/{init,join,configure,repo}.ts` and
      `_shared.ts` read in full, in their current state (not inferred
      from the roadmap's pre-INC-01/02 description).
- [x] Exact current questionnaire/flag values confirmed directly from
      code (`functionalProfile`, `overlay`, `adapterTarget`), not
      assumed.
- [x] Gap analysis table produced covering all 6 questionnaire
      dimensions (identity/repos/roles/adapters/tools/bootstrap) against
      current-state code, each row citing the specific file/function
      that grounds the claim.
- [x] Concrete determination of whether `init.ts` needs to start asking
      "where is your SDD/spec repo," grounded in what `init.ts` and
      `repo.ts` actually do today.
- [x] Explicit recommendation on `schemaVersion: 2` re-enablement scope
      (in this increment vs. a separate task), with rationale tied to
      AGENTS.md's preference for small, focused increments.
- [x] Explicit confirmation of whether `init` already writes an
      `AGENTS.md` (with file/line evidence), shaping the next role's
      brief as "extend/verify" vs. "create from scratch."
- [x] No files changed under `Axiom/`.
- [x] Recorded in `Axiom.Spec` per `Axiom.SDD/AGENTS.md`.

## Open questions

None blocking. One flag for the roadmap's own role sequencing is raised
in "Recommendation on subagent sequencing" below — this is a finding to
surface, not a question requiring a decision before proceeding.

## Assumptions

- The roadmap's original INC-03 diff hypothesis (installer asks
  `functionalProfile`/`overlay`/`adapterTarget`, not identity/repos/
  roles/adapters/tools/bootstrap) was written before INC-01/INC-02
  landed and needed re-confirmation against current code — done in this
  audit; the hypothesis about the questionnaire shape holds, but several
  of its specifics about `init`'s repo-awareness are now stale (see
  Findings).
- "Reconcile" in this increment's title means: identify what already
  covers each new-doc dimension, what's missing, and what changed
  because of INC-01/INC-02 — not implement anything yet.

## Implementation notes

Audit-only. No files were changed under `Axiom/packages/`, `Axiom/apps/`,
or `Axiom.SDD/` for this increment.

### Files read in full

- `Axiom/packages/installer/src/installer.ts`, `registry.ts`,
  `profiles-loader.ts`, `persist.ts`, `types.ts`, `index.ts`
- `Axiom/packages/installer/tests/installer.test.ts`
- `Axiom/packages/install-profiles/src/composer.ts`, `types.ts`,
  `constants.ts`, `role-aliases.ts`, `overhead.ts`, `index.ts`
- `Axiom/apps/cli/src/commands/init.ts`, `join.ts`, `configure.ts`,
  `repo.ts`, `_shared.ts`
- `Axiom/packages/project-resolution/src/resolver.ts`
- `Axiom/packages/config-validation/src/schemas.ts`
- `Axiom/packages/user-workspace/src/registry-types.ts`
- Both external decision documents (source doc §13.2, full addendum)

### Finding 1 — `@axiom/installer`'s `registry.ts` is a different file than `@axiom/user-workspace`'s `registry.ts`

Confirmed. `Axiom/packages/installer/src/registry.ts` exports
`GENERATED_FILES_BY_TARGET` (a static map of adapter target ID ->
generated file paths, e.g. `'opencode': ['.opencode/AGENTS.md',
'.opencode/skills-lock.yaml']`) and `EXTERNAL_DEPS_BY_CAPABILITY`. It has
nothing to do with the project registry (`~/.axiom/projects.yml`), which
lives in the unrelated `@axiom/user-workspace` package's own
`registry.ts`. Both files are named `registry.ts` but serve unrelated
purposes; this is a naming collision, not a shared module. No action
needed beyond flagging it so future readers do not conflate the two.

### Finding 2 — The current questionnaire shape is exactly the roadmap's hypothesis, values confirmed

`@axiom/install-profiles/src/constants.ts` and `types.ts` confirm the
exact literal values:

- `functionalProfile`: `'product-owner' | 'builder'`
  (`FUNCTIONAL_PROFILES` in `constants.ts`).
- `overlay`: `'local-only' | 'standard' | 'enterprise'`
  (`OPERATIONAL_OVERLAYS` in `constants.ts`).
- `adapterTarget`: `AdapterTargetId`, with 8 concrete values confirmed in
  `installer/registry.ts`'s `GENERATED_FILES_BY_TARGET` keys: `opencode`,
  `copilot-vscode`, `claude-code`, `antigravity`, `visual-studio-2026`,
  `cursor`, `github-copilot`, `litellm`.

The "10-step composer" referenced in the roadmap is
`resolveInstallProfile` in `composer.ts` — a 10-internal-step **pure
function** (validate triple -> select capabilities -> resolve discovery
profile -> resolve providers -> resolve gateway expectation -> compute
token overhead -> attach compliance risk -> build degradation model),
not a 10-question interactive wizard. There is no interactive wizard at
all today: `init.ts`'s `registerInit` reads flags
(`--profile`/`--overlay`/`--target`/`--layout`/`--name`) via `commander`,
with `-y/--yes` accepted as a flag but not actually gating any prompt
logic in the read code (no `inquirer`- or `prompts`-style interactive
step was found in `init.ts`, `join.ts`, `configure.ts`, or `repo.ts`).
This confirms the roadmap's diff: today's flow is a capability/profile
selection via flags, not the new docs' identity/repos/roles/adapters/
tools/bootstrap questionnaire.

### Finding 3 — `init.ts` does not ask "where is your SDD/spec repo," and per current `repo.ts` behavior this is correctly out of `init`'s scope

`init.ts` hardcodes `DEFAULT_REPO_ROLE = 'sdd'` and registers exactly one
repo (the cwd) under that role when it calls `addProjectV2`. There is no
flag or prompt asking about a separate SDD repo, SPEC repo, or code
repos. This matches the roadmap's original diff claim.

However, the roadmap's original framing ("does `init` need to start
asking this now that INC-01 landed multi-repo registry support") is
answered by directly reading `repo.ts`: `runRepoAttach` (in
`apps/cli/src/commands/repo.ts`) already implements exactly the
"add another repo under an explicit role to an existing/new project"
operation, accepting `--role` (default `'sdd'`, but overridable to
`spec`/`code`/anything) and calling the same `addProjectV2` that `init`
uses. This is a real, working, tested command surface today — not a gap.

Conclusion: **`init` staying single-repo-first, with multi-repo growth
handled by `axiom repo attach --role <role>`, is the correct design
given what already exists** — it is not an omission. The registry
*schema* (INC-01) already supports multiple repos per project; the
*command* that adds subsequent repos (`repo attach`) already exists;
what's missing is only that `init`'s interactive/flag surface never asks
the identity/roles/adapters/tools/bootstrap questions from source doc
13.2 at all — that gap is about the *questionnaire depth*, not about
repo-count support. Reframing `init` as "ask about SDD+SPEC+code repos
inline during init" would duplicate `repo attach`'s job and contradicts
the parent roadmap's own instruction not to introduce speculative
architecture beyond what's asked. The recommended target: keep `init`
single-repo (with an optional prompt, gated behind future UX work, to
suggest running `repo attach` for a SPEC/SDD split), and treat the
richer questionnaire as the shape `init` asks for *its own* repo's
identity/role, not as a multi-repo collector.

### Finding 4 — `init` does not write `AGENTS.md`; `configure` writes adapter-specific docs conditionally, not a canonical `AGENTS.md`

Grep across `apps/cli/src/commands/*.ts` for `AGENTS.md` found exactly
one match: `configure.ts`, and only indirectly — `configure.ts` calls
`installProfile` (from `@axiom/installer`), which looks up
`GENERATED_FILES_BY_TARGET[adapterTarget]`. For `adapterTarget:
'opencode'`, that list includes `.opencode/AGENTS.md`; for `antigravity`,
`.antigravity/AGENTS.md`. These are adapter-scoped, tool-specific files
(the `opencode`/`antigravity` CLI/IDE's own instruction file convention),
generated only when `configure` runs with that specific adapter target
selected at `init` time. There is **no root-level, canonical `AGENTS.md`**
written by `init` or `configure` in the source-doc sense (addendum
section 3: "AGENTS.md será el contrato canónico portable por repo," a
single upstream file that adapters derive from). What exists today is
the inverse: `installer/registry.ts`'s static map treats `AGENTS.md` as
one of several *per-adapter output filenames*, not as the canonical
upstream input.

Conclusion for the next role's brief: `init`/`configure` do **not**
already write a canonical repo-root `AGENTS.md`. The docs-skills-writer's
task is "create," not "extend/verify" — but scoped narrowly: introduce
one canonical `AGENTS.md` per repo (per addendum section 3) as a new
artifact, without assuming any existing renderer already produces it.
The existing `.opencode/AGENTS.md`/`.antigravity/AGENTS.md` per-adapter
outputs are a different, adapter-local concept and should not be
conflated with the new canonical `AGENTS.md`; whether they eventually
derive from it is a design question for a later increment (the roadmap's
own INC-05 already scopes exactly this: "AGENTS.md as canonical contract
+ adapter reconciliation").

### Finding 5 — `init.ts`'s `schemaVersion: 2` blocker comment is now stale; the blocking condition it describes no longer holds

`init.ts` (lines ~206-238) contains a substantial comment explaining that
emitting `axiom.yaml` with `schemaVersion: 2` was attempted and reverted
because `@axiom/project-resolution`'s `resolveProject` "SÓLO lee
`config.project.name` (shape v1)" and would return `status: 'ambiguous'`
for a v2-shaped `axiom.yaml`.

Reading `packages/project-resolution/src/resolver.ts` directly
(current state, same commit `4ba8abe` that landed registry v2 and wrote
this comment) shows this is **no longer true**: `resolveProject` calls
`validateAxiomYamlContent`, branches on `validation.version === 1` vs.
`2`, and has a fully implemented `resolveFromV2` path that reads
`projectId`/`name`/`repoId`/`role`/`paths`/`mode` and normalizes them to
the same `ProjectResolution` shape v1 produces. Both `resolveFromV1` and
`resolveFromV2` were touched in the same commit as `init.ts`'s revert
comment (confirmed via `git log --oneline -- <file>` on both files:
identical commit `4ba8abe`). The most likely explanation is a
within-session sequencing issue — the resolver fix landed, but `init.ts`'s
comment (and its behavior) were not updated to reflect it, or the
revert decision was made deliberately even after the resolver fix for
reasons not evident in the comment (e.g. wanting a broader validation
pass across doctor/downstream consumers before flipping the emission
default). Either way, **the specific blocking condition documented in
`init.ts`'s comment (`resolveProject` cannot read v2) is factually false
against the current codebase.**

### `schemaVersion: 2` re-enablement — scope recommendation

**Recommendation: keep it as a separate, smaller follow-up task, not
folded into this increment's implementation phase — but flag it as
unblocked, not blocked, in that follow-up's brief.**

Rationale:

1. AGENTS.md's guardrails (`Axiom.SDD/AGENTS.md`, "Keep changes small
   and focused," "Do not modify unrelated files") favor a narrow,
   single-purpose change: flipping `init.ts`'s emission default touches
   exactly one file's `buildAxiomYaml`-equivalent logic plus its own
   tests, and is independently testable/revertable from the rest of
   INC-03's installer-questionnaire work.
2. This increment (INC-03) is scoped to the *questionnaire* dimensions
   (identity/repos/roles/adapters/tools/bootstrap) per the parent
   roadmap's own title. The `schemaVersion` emitted by `buildAxiomYaml`
   is an orthogonal concern: whether `init` emits v1 or v2 shape does
   not change what questions are asked, only what gets written to disk
   afterward.
3. Folding it in would require the next implementing role
   (cli-implementer, per the roadmap) to touch two independent surfaces
   in one pass (new questionnaire prompts + schema version cutover),
   raising review risk without a corresponding benefit — the roadmap's
   own "Automatic Mode Gatekeeper" / "Review Workload Guard" spirit
   (small, reviewable diffs) argues for splitting these.
4. It is empirically unblocked now (Finding 5), so there is no reason to
   wait for INC-03 to complete before scheduling it — it can run before,
   after, or in parallel with INC-03's questionnaire work, as its own
   increment (e.g. `INC-20260702-init-schemaversion2-cutover`).

Concretely: recommend opening a small, dedicated increment whose only
scope is (a) re-verify Finding 5's claim holds under the full doctor/
downstream consumer set (MC-001, BC-001, `project-resolution` consumers
beyond the CLI — a broader check than this audit performed), (b) flip
`buildAxiomYaml` to emit v2 shape (with `--schema-version` escape hatch
if a compatibility path is wanted), (c) update/add tests, (d) remove the
now-stale comment. This is a natural `migration-engineer` -> `schema-
writer`/`cli-implementer` -> `validator-reviewer` sequence, same pattern
as INC-01.

## Gap analysis: questionnaire dimensions vs. current installer state

| Dimension | Source doc 13.2 / addendum ask | Current state (file/function evidence) | Gap |
|---|---|---|---|
| **Identity** | Project name, `projectId`, new-vs-existing, migrate-from-legacy-SDD | `init.ts`: `--name` flag (default: cwd basename), `validateProjectName` (DNS-label regex). No `projectId` distinct from `name` in v1 emission (v2 schema has it, but v1 is what's emitted — Finding 5). No new-vs-existing prompt: `init` always creates; re-running requires `--force`. No migrate-from-legacy-SDD path (`bootstrap-from-legacy-sdd`, INC-19 territory) exists anywhere in the audited files. | Partial. Name exists; projectId/new-or-existing/migrate-from-legacy are missing, migrate-from-legacy explicitly deferred to a later phase (INC-19) per the roadmap, not this increment's job. |
| **Repos** | Which is SDD, which is SPEC, paths, code repos, kind+stack per repo | `init.ts` registers exactly one repo (cwd) under hardcoded role `sdd` — no SDD-vs-SPEC-vs-code selection at `init` time. `repo.ts`'s `runRepoAttach` DOES support attaching additional repos under an explicit `--role` (`sdd`/`spec`/`code`/etc.) to the same project id, confirmed working and tested. No `kind`/`stack` fields exist anywhere in the v2 registry (`RegistryRepoEntry` only has `role`+`path`) or in `axiom.yaml` v2 (`AxiomYamlSchemaV2` has no `kind`/`stack` field either — confirmed in `schemas.ts`). | Partial, split by command: `init` intentionally stays single-repo (see Finding 3 — correct by design, not a gap to close inside `init`); `repo attach` already covers "add repo with role" but lacks `kind`/`stack` metadata the source doc wants per repo. |
| **Roles** | Active roles list, repos associated per role, default role suggestions (PO/Analyst/Architect/Backend/Frontend/QA/DevOps/Orchestrator) | `install-profiles`' `functionalProfile` (`product-owner`/`builder`) is a coarse 2-value axis, not the 8-role list from source doc 13.2. `role-aliases.ts` supports `analista`/`arquitecto` as CLI-facing aliases for the same 2 canonical values — a naming bridge, not additional roles. No per-role repo-association model exists. | Large gap. The existing 2-value functional-profile axis serves a different purpose (capability/token-budget selection) than the source doc's 8-role-with-repo-associations model; these are not the same concept and should not be conflated when scoping future work. |
| **Adapters** | OpenCode, Claude Code, GitHub Copilot, Antigravity, Cursor, Gemini CLI, other | `adapterTarget` covers `opencode`, `claude-code`, `github-copilot`, `copilot-vscode`, `antigravity`, `cursor`, `visual-studio-2026`, `litellm` (8 values, confirmed in `installer/registry.ts`). Gemini CLI is absent. `init`/`configure` only ever materialize ONE adapter target per project (a single `--target` flag), not a multi-select of adapters as the source doc implies ("Adapters:" as a checklist, not a single choice). | Partial. Coverage of tool identities is broader than the source doc's list (minus Gemini CLI), but the selection model (single target vs. multi-select) differs structurally. |
| **Tools/MCPs** | Axiom project MCP, Serena per repo, Context7, Azure DevOps/Jira, other MCPs | `EXTERNAL_DEPS_BY_CAPABILITY` (`installer/registry.ts`) computes `serena`/`codegraph`/`sdd-mcp-server`/`spec-mcp-broker` as *derived* external dependencies from enabled capabilities — not a user-facing selection step. No `mcp.yml`-equivalent manifest, no Context7 or Azure DevOps/Jira presence found in the audited files (Azure DevOps plugin exists elsewhere per the parent roadmap's INC-16, not in this scope). | Large gap. Tool/MCP selection today is a derived side-effect of capability selection, not an explicit "which MCPs do you want" question; no per-project `mcp.yml` manifest exists yet (parent roadmap's INC-13/15 territory). |
| **Bootstrap** | Yes/no, source (code vs legacy SDD), exhaustiveness level (0-4) | No bootstrap step exists anywhere in `init`/`join`/`configure`/`repo`. `@axiom/document-bootstrap` (referenced by `configure.ts` for `writeCopilotInstructions`) renders already-known variables into one target file; it does not analyze code or offer exhaustiveness levels. | Full gap, explicitly out of this increment's scope — matches parent roadmap's INC-18/INC-19 (Phase F), not INC-03. |

## Recommendation on subagent sequencing

The parent roadmap's planned INC-03 subagent order is:
migration-engineer -> docs-skills-writer -> registry-engineer ->
cli-implementer -> validator-reviewer.

Based on this audit, flagging (not silently reordering) one sequencing
concern: **docs-skills-writer's `AGENTS.md` work has no dependency on
`init`'s questionnaire changes and could run in parallel with, or before,
the registry-engineer/cli-implementer steps**, since Finding 4 confirms
`AGENTS.md` generation is a net-new artifact untouched by any existing
installer code path — it does not need the new questionnaire's repo/role
data to produce a first version (a minimal `AGENTS.md` per addendum
section 18.1/18.2's structural-integrity + separation-of-concerns
boilerplate needs only `projectName`/`role`, both already available
today). Running docs-skills-writer first (as the roadmap already
proposes) is reasonable, but it should be scoped narrowly enough that it
does not block on registry-engineer's output. No reordering performed;
this is surfaced for the next role/orchestrator to confirm before
starting docs-skills-writer's work.

## Validation

`No validation command was found. Performed best-effort validation by inspecting changed files and checking consistency against the requested behavior.`

No code was changed in this increment, so there is nothing to run a
build/test/lint command against. Best-effort validation performed:

- Every claim in the Findings and gap-analysis table above is grounded
  in a specific file read in full during this session (listed under
  "Files read in full"), not inferred from file/package names.
- Finding 5's claim (`resolveProject` already handles v2) was
  cross-checked against `git log --oneline` for both `resolver.ts` and
  `init.ts`, confirming both were touched in the same commit (`4ba8abe`),
  which supports the "stale comment, not stale code" conclusion rather
  than a timeline where the resolver fix came later.
- The exact enum values quoted (`functionalProfile`, `overlay`,
  `adapterTarget`) were cross-checked between `install-profiles/
  constants.ts`, `installer/registry.ts`'s map keys, and
  `installer/tests/installer.test.ts`'s fixture, which independently
  lists the same 8 adapter targets.
- Did not run `Axiom`'s actual test suite (`npm test`) because no source
  code was modified; running it would only assert pre-existing state,
  per the same rationale the parent roadmap increment used.

## Result

Produced a full audit of the current Axiom installation wizard flow
against the target questionnaire (source doc 13.2 + addendum). Key
findings: (1) the roadmap's original questionnaire-shape diff hypothesis
holds, with exact values now confirmed from code; (2) `init` staying
single-repo-first is correct by design given that `repo attach` already
covers multi-repo growth — this is not a gap needing correction inside
`init`; (3) `init`/`configure` do not write a canonical `AGENTS.md` today
(only adapter-scoped files with that literal filename for `opencode`/
`antigravity` targets) — the docs-skills-writer's task is create, not
extend; (4) the `schemaVersion: 2` re-enablement blocker documented in
`init.ts` is factually stale against the current `project-resolution`
resolver and should be handled as its own small follow-up increment, not
folded into this one; (5) the Roles and Tools/MCPs dimensions have the
largest gaps against the source docs, while Identity/Repos/Adapters have
partial coverage under different existing concepts that should not be
conflated with the new docs' model.

## General spec integration

No integration into `Axiom.Spec/general-spec.md` was performed — this
file does not exist in this repo (confirmed absent, consistent with the
parent roadmap increment's own note). The closest equivalents are
`Axiom.Spec/specs/00_Resumen_Ejecutivo.md` through `08_Glosario.md`,
which were not modified: they describe intent at a level this audit's
findings do not change. One item worth flagging for whoever eventually
consolidates stable knowledge: Finding 5 (the stale `schemaVersion: 2`
blocker comment in `init.ts`) is itself a piece of implementation-detail
knowledge that belongs in `Axiom.SDD`-side code comments once the
follow-up increment fixes it, not in `Axiom.Spec` — no action needed
here beyond this increment's own record.

## Next step recommendation

Proceed with **docs-skills-writer** next, scoped narrowly per Finding 4:
create (not extend) a minimal canonical `AGENTS.md` template per repo,
covering addendum section 18.1 (structural integrity rules) and 18.2
(separation of concerns), parameterized only on data already available
today (`projectName`, `role`, `layout`). Do not block this on the
Roles/Tools-MCPs gaps identified above (Phase D/E territory per the
parent roadmap) or on the richer questionnaire (registry-engineer/
cli-implementer's later steps in this same INC-03) — per the sequencing
note above, this can run in parallel with or before those.

Separately, recommend opening a small dedicated follow-up increment
(e.g. `INC-20260702-init-schemaversion2-cutover`) for the `schemaVersion:
2` re-enablement in `init.ts`, per the rationale in that section — do not
fold it into this increment's remaining implementation phases.
