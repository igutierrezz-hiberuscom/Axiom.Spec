# Increment: AGENTS.md canonical contract vs adapter reconciliation (audit)

Status: pending
Date: 2026-07-02

## Goal

Audit whether the 6 tool adapters under `Axiom/packages/adapters/*`
derive from, or diverge from, the new canonical repo-root `AGENTS.md`
introduced by `INC-20260702-installer-wizard-reconcile-agents-md`, per
the addendum §3 requirement ("adapters específicos de herramienta se
generarán a partir de AGENTS.md y de los manifests/índices de Axiom...
no mantener instrucciones divergentes por herramienta si pueden
derivarse del contrato común"). This is the `migration-engineer` role
for the parent roadmap's INC-05
(`Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/README.md`).
Audit only — no code changes.

## Context

Parent roadmap INC-05 originally assumed no canonical `AGENTS.md`
pipeline existed yet. That framing is now stale: INC-03's
docs-skills-writer step already built one
(`Axiom/packages/document-bootstrap/src/canonical-agents-md.ts` —
`renderCanonicalAgentsMd`/`writeCanonicalAgentsMd`), wired into
`Axiom/apps/cli/src/commands/init.ts`, writing a repo-root `AGENTS.md`
with source-doc §18.1/§18.2 boilerplate verbatim plus a minimal
project-identity header (`projectName`, optional `role`/`layout`).
Full details in
`Axiom.Spec/specs/increments/INC-20260702-installer-wizard-reconcile-agents-md/README.md`.

Given that, this increment's real scope is narrower than the original
roadmap text: not "does a canonical `AGENTS.md` exist" (yes) but "do
the 6 adapter generators derive from it, or read something else
entirely with no relationship to it."

## Scope

- Read `Axiom/packages/document-bootstrap/src/*` in full (already done
  in this pass): confirms the OLD single-target pipeline
  (`renderCopilotInstructions`/`resolveVariables`/`writeCopilotInstructions`,
  scoped to `.github/copilot-instructions.md`, keyed on
  `ResolvedVariables` — `project.name`/`project.id`/`mcp.allowlist`/
  `support.level`/`target.id`) and the NEW canonical-`AGENTS.md`
  pipeline (`renderCanonicalAgentsMd`/`writeCanonicalAgentsMd`, keyed
  on `CanonicalAgentsMdIdentity` — `projectName`/`role`/`layout`) as
  two independent, non-overlapping code paths in the same package.
- Read all 6 adapter packages under `Axiom/packages/adapters/*`
  (`opencode`, `claude-code`, `github-copilot`, `vscode`, `cursor`,
  `litellm` — confirmed exact directory/package names) in full:
  `generator.ts`, `types.ts`, and their renderer modules
  (`agents-md.ts`, `instructions.ts`, `settings.ts`, `config.ts`).
- Read `Axiom/packages/installer/src/registry.ts`
  (`GENERATED_FILES_BY_TARGET`, `EXTERNAL_DEPS_BY_CAPABILITY`) — the
  static generated-files map.
- Read `Axiom/apps/cli/src/commands/sync.ts` and `configure.ts` to see
  how/when adapter generation is actually invoked.
- Read `Axiom/packages/doctor/src/checks.ts`'s `TC-009` check
  (`runAdapterRuntimeCoverageCheck`).
- Produce a per-adapter diff table: does each adapter's generated
  output overlap in content with canonical `AGENTS.md`, or is it
  genuinely adapter-specific (tool config, not portable instructions)?
- Recommend a minimal reconciliation approach (or explicitly recommend
  none) consistent with `Axiom.SDD/AGENTS.md`'s bootstrap limits.
- Recommend the next role in the INC-05 subagent sequence.

## Non-goals

- No implementation of any reconciliation mechanism.
- No changes to `@axiom/document-bootstrap`, any `@axiom/adapters-*`
  package, `@axiom/installer`, or `@axiom/doctor`.
- No re-litigation of INC-03's canonical `AGENTS.md` template content
  (settled; treated as given input here).
- No decision on `schemaVersion: 2` cutover (tracked separately,
  unrelated to this audit).

## Acceptance criteria

- [x] `Axiom/packages/document-bootstrap/src/*` read in full; old vs
      new pipeline contrasted explicitly.
- [x] All 6 adapter packages read in full (`generator.ts` + renderer
      module + `types.ts` each).
- [x] `Axiom/packages/installer/src/registry.ts` read in full;
      `GENERATED_FILES_BY_TARGET` confirmed as the static source of
      truth for which files each target emits.
- [x] Invocation sites (`sync.ts`, `configure.ts`) read to confirm
      when/how adapter generation actually runs today.
- [x] `TC-009` (`runAdapterRuntimeCoverageCheck`) read and its exact
      current behavior documented (package-shape existence check, not
      a content/consistency check).
- [x] Per-adapter diff table produced, classifying each adapter as
      "genuine overlap with canonical AGENTS.md" vs "no overlap /
      adapter-specific only."
- [x] A concrete, minimal reconciliation recommendation given (or
      explicit "no action needed" per adapter), consistent with
      `Axiom.SDD/AGENTS.md` bootstrap limits.
- [x] Next-role recommendation given and justified against what was
      actually found (confirm or revise skipping docs-skills-writer).
- [x] No files changed under `Axiom/packages/` or `Axiom/apps/`.
- [x] Recorded in `Axiom.Spec` per `Axiom.SDD/AGENTS.md`.

## Findings

### 1. Document-bootstrap: two independent pipelines, no shared source

`@axiom/document-bootstrap` (`Axiom/packages/document-bootstrap/src/`)
contains two functionally disjoint pipelines that happen to share only
the idempotency/atomic-write plumbing:

| | OLD pipeline | NEW pipeline |
|---|---|---|
| Entry point | `writeCopilotInstructions` (`writer.ts`) | `writeCanonicalAgentsMd` (`canonical-agents-md.ts`) |
| Pure renderer | `renderCopilotInstructions` (`renderer.ts`) | `renderCanonicalAgentsMd` (`canonical-agents-md.ts`) |
| Input shape | `ResolvedVariables` (`project.name`, `project.id`, `mcp.allowlist`, `support.level`, `target.id`, optional `target.docs.url`) resolved by `resolveVariables` (`variables.ts`) from `ProductManifest`/`ProvidersYaml`/`LocalOverlay`/`axiom.yaml` fallback | `CanonicalAgentsMdIdentity` (`projectName`, optional `role`, optional `layout`) passed directly by the caller |
| Output target | `.github/copilot-instructions.md` (template-driven, `{{placeholder}}` substitution) | `<projectRoot>/AGENTS.md` (hardcoded verbatim §18.1/§18.2 blocks + identity header, no template file) |
| Wired into | Nothing today in `apps/cli` calls `writeCopilotInstructions` directly for the `.github/copilot-instructions.md` path — `configure.ts` calls it only for `adapterTarget === 'copilot-vscode' \| 'github-copilot'` (see Finding 3) | `init.ts`, additive/best-effort, right after `axiom.yaml` is written |
| Idempotency | `classifyAndPreserve` (`idempotency.ts`) | Same `classifyAndPreserve` (reused) |

They share the `AXIOM:GENERATED`/`TEAM:CUSTOM` marker convention and
the atomic tmp+rename write pattern, but **no data flows from one to
the other**. `writeCanonicalAgentsMd` does not read anything
`resolveVariables` produces, and vice versa. There is no "canonical
AGENTS.md is upstream, copilot-instructions is downstream" relationship
in this package today — they are parallel, independently-invoked
writers.

### 2. Per-adapter diff (`Axiom/packages/adapters/*`)

All 6 adapters read the **same single input shape** today:
`ResolvedInstallProfile` (from `@axiom/install-profiles`), obtained via
`readInstallProfileSync(rootPath, projectName)` reading
`.sdd/<projectName>/install-profile.json` — **not** `axiom.yml`
directly, and **not** the canonical `AGENTS.md`. `ResolvedProfile`
exposes `enabledCapabilities: string[]` and `adapterTarget: string`;
every adapter's renderer sorts `enabledCapabilities` alphabetically and
renders it as a bullet list, optionally alongside adapter-specific
sections.

| Adapter package | Generated file(s) | Data source | Content overlap with canonical `AGENTS.md` | Verdict |
|---|---|---|---|---|
| `opencode` (`@axiom/adapters-opencode`) | `.opencode/AGENTS.md`, `.opencode/skills-lock.yaml` | `ResolvedInstallProfile` + `OpencodeSkill[]` | `.opencode/AGENTS.md` renders `## Generated portable entries` → `### Capabilities` (from `enabledCapabilities`) + `### Skills` (from `skills[].portableEntry`). **No** structural-integrity or separation-of-concerns content; no project-identity header (`projectName`/`role`). Genuinely different content domain (capability/skill routing, not repo governance rules). | **No overlap.** Named `AGENTS.md` but a different artifact by content, not just by path. No reconciliation needed for content; only a naming/documentation clarification is arguably useful (see Recommendation). |
| `claude-code` (`@axiom/adapters-claude-code`) | `.claude/AGENTS.md` (note: `GENERATED_FILES_BY_TARGET` in `registry.ts` documents `.claude/CLAUDE.md`, but the actual generator writes `.claude/AGENTS.md` — see Finding 4, a pre-existing inconsistency, not introduced by this audit) | `ResolvedInstallProfile` + `ClaudeCodeSkill[]` | Same `## Generated portable entries` block as opencode, plus a `## Routing note` (single-mode). Same verdict as opencode: capability/skill routing, not governance rules. | **No overlap.** Same reasoning as opencode. |
| `github-copilot` (`@axiom/adapters-github-copilot`) | `.github/copilot-instructions.md` | `ResolvedInstallProfile` (+ optional `instructions: string[]`) | Renders `# Copilot instructions for <projectName>` → `## Capabilities` (alphabetical bullets) → optional `## Additional instructions` → `## Notes` (fixed provenance bullet). No structural-integrity/separation-of-concerns content, no marker-based preservation (full overwrite each run). | **No overlap.** Genuinely different content (capabilities list + provenance note), not portable governance rules. |
| `vscode` (`@axiom/adapters-vscode`) | `.vscode/settings.json`, `.vscode/extensions.json` | `ResolvedInstallProfile` + optional `extensions: string[]` | Both outputs are JSON, not Markdown. `settings.json` is `{}` in the MVP; `extensions.json` is a recommendations list. No prose content at all. | **No overlap — structurally cannot overlap** (JSON config, not instructions). |
| `cursor` (`@axiom/adapters-cursor`) | `.cursor/settings.json`, `.cursor/AGENTS.md` | `ResolvedInstallProfile` | `settings.json` is `{}` (JSON, no overlap). `.cursor/AGENTS.md` renders `# AGENTS.md (Cursor)` → `## Generated by Axiom` → `### Capabilities` (alphabetical) + a fixed disclaimer. No structural-integrity or separation-of-concerns content. | **No overlap.** Same reasoning as opencode/claude-code: capability list, not governance rules. |
| `litellm` (`@axiom/adapters-litellm`) | `litellm.config.json` | `ResolvedInstallProfile` (+ optional `models`) | Pure JSON LiteLLM router config (`model_list`, `litellm_settings`, `general_settings`). No prose at all. | **No overlap — structurally cannot overlap.** |

**Result: zero of the 6 adapters currently duplicate the §18.1
("Axiom structural integrity rules") or §18.2 ("Separation of
concerns") boilerplate that canonical `AGENTS.md` renders.** Every
adapter that emits a file literally named `AGENTS.md`
(`opencode`, `claude-code`, `cursor`) emits a **capability/skill
routing block**, which is a different content domain from the
governance/structural rules in canonical `AGENTS.md`. The addendum §3
concern ("no mantener instrucciones divergentes por herramienta si
pueden derivarse del contrato común") does not currently apply to any
of the 6 adapters, because none of them re-state or diverge from
content that canonical `AGENTS.md` already owns — they simply don't
touch that content domain at all.

This directly narrows the parent roadmap INC-05 diff's framing
("today the canonical source for generated files is `axiom.yml` +
`install-profile.json` + per-adapter generator, not `AGENTS.md`... a
shim... avoids a hard cutover"): there is no divergent/duplicated
*content* to reconcile. The adapters' inputs (`ResolvedInstallProfile`)
and canonical `AGENTS.md`'s input (`CanonicalAgentsMdIdentity`) are
disjoint by design — one is a capability/skill list, the other is
governance boilerplate + project identity.

### 3. Invocation timing: two separate call sites, one inconsistency worth flagging

- `apps/cli/src/commands/sync.ts` (`materializeAdapterOutputs`) is the
  generic dispatcher: a `switch (adapterTarget)` that calls all 6
  generators uniformly, always reading `ResolvedInstallProfile` via
  `readRequiredInstallProfile`. This is the primary, complete
  invocation path, run at `axiom sync` time.
- `apps/cli/src/commands/configure.ts` **also** calls
  `generateOpencodeConfig` directly (inline, not via `sync.ts`'s
  dispatcher) when `adapterTarget === 'opencode'`, and **separately**
  calls the OLD `writeCopilotInstructions` pipeline (not
  `generateCopilotConfig`) when `adapterTarget` is `'copilot-vscode'`
  or `'github-copilot'`. This means `configure.ts` mixes the NEW
  adapter-package pipeline (for opencode) with the OLD
  `ResolvedVariables`-based pipeline (for copilot targets) in the same
  function, while `sync.ts` uses the NEW adapter-package pipeline
  (`generateCopilotConfig`) for copilot uniformly. This is a
  pre-existing inconsistency between `configure.ts` and `sync.ts`,
  **unrelated to canonical `AGENTS.md`** (it predates INC-03's
  canonical-`AGENTS.md` work) — flagged here because it is adjacent
  ground truth the next role should not confuse with an
  `AGENTS.md`-adapter reconciliation gap. It is out of scope for this
  increment.
- Canonical `AGENTS.md` (`writeCanonicalAgentsMd`) is invoked only from
  `init.ts`, once, at project creation. It is never re-invoked by
  `sync.ts` or `configure.ts`. This means canonical `AGENTS.md` and the
  6 adapters are generated at different lifecycle moments today
  (`init` vs `sync`/`configure`), which is consistent with canonical
  `AGENTS.md` being a one-time identity artifact rather than a
  per-sync-cycle regenerated one — not itself a defect, but relevant
  context if a future increment wants canonical `AGENTS.md` to be an
  upstream input re-read on every adapter generation pass (it would
  need to also run at `sync`/`configure` time, not just `init`).

### 4. Pre-existing naming inconsistency (adjacent finding, not this increment's scope)

`Axiom/packages/installer/src/registry.ts`'s `GENERATED_FILES_BY_TARGET`
declares `'claude-code': ['.claude/CLAUDE.md', '.claude/skills-lock.yaml']`,
but `@axiom/adapters-claude-code`'s actual `generator.ts` writes
`.claude/AGENTS.md` and does not emit a skills lockfile at all (no
`skills-lock.ts` module exists in that package, confirmed by the file
listing). This is a real drift between the static registry map and the
actual runtime behavior, but it is orthogonal to the `AGENTS.md`
canonical-contract question this increment audits — flagged for
whichever role next touches `@axiom/installer`'s registry or
`@axiom/adapters-claude-code`, not addressed here.

### 5. TC-009 current behavior (`Axiom/packages/doctor/src/checks.ts`)

`runAdapterRuntimeCoverageCheck` (`TC-009`, category `adapters`)
verifies **package shape only**: for each of the 6
`ADAPTER_PACKAGES` (`opencode`, `claude-code`, `github-copilot`,
`vscode`, `cursor`, `litellm`), it checks that
`packages/adapters/<target>/src/generator.ts` and
`packages/adapters/<target>/dist/index.js` exist on disk. It does
**not** inspect generated file *content*, does not check consistency
with canonical `AGENTS.md`, and does not check the
`GENERATED_FILES_BY_TARGET` registry against actual generator output
(so it would not catch Finding 4's `CLAUDE.md`/`AGENTS.md` drift
either). This confirms the parent roadmap's own note that
validator-reviewer should "extend it, do not replace it" — but per
Finding 2, there is currently no consistency invariant to enforce
between adapters and canonical `AGENTS.md`, because there is no
content overlap to keep consistent. TC-009 does not need a new check
today; it would only need one if/when a future increment makes an
adapter derive prose content from canonical `AGENTS.md`.

## Recommendation

**No reconciliation code is needed right now.** The parent roadmap's
proposed default shim ("adapters keep reading their current inputs,
plus a new `AGENTS.md` render step that stays consistent with them")
does not apply because there is nothing to keep consistent — the 6
adapters and canonical `AGENTS.md` occupy disjoint content domains
(capability/skill routing + tool config vs. repository governance
rules + project identity). Forcing adapters to read from or derive
content out of canonical `AGENTS.md` today would mean inventing new
adapter content that does not currently exist, which is exactly the
speculative-architecture pattern `Axiom.SDD/AGENTS.md` prohibits
("Do not create speculative architecture", "Do not introduce ... unless
explicitly requested").

This finding **overturns** parent-roadmap INC-05's "potentially
breaking" framing: after reading all 6 adapters directly (the roadmap
draft had not yet done this line-by-line read when it flagged INC-05),
the correct minimal action is:

1. **No code change to any of the 6 adapters.** None of them currently
   duplicate or diverge from canonical `AGENTS.md` content.
2. **Document the finding** (this file) so a future increment that
   *does* want tool-specific instructions to route through canonical
   `AGENTS.md` (e.g. if `opencode`/`claude-code`/`cursor`'s
   `AGENTS.md`-named outputs are later redesigned to also carry the
   structural-integrity/separation-of-concerns content, or to link
   back to the canonical file instead of being silent about it) has a
   concrete, evidence-based starting point instead of re-auditing from
   scratch.
3. **One low-cost, genuinely optional improvement** worth flagging to
   the user, not prescribing: `opencode`/`claude-code`/`cursor`'s
   `AGENTS.md`-named files could each add a single line pointing back
   to the repo-root canonical `AGENTS.md` (e.g. "See the repo-root
   `AGENTS.md` for structural rules that apply regardless of tool
   adapter"), so a reader landing in `.opencode/AGENTS.md` knows the
   canonical file exists. This is a one-line addition per adapter, not
   a pipeline change, and is explicitly **not** required by any
   acceptance criterion here — flagged as a future consideration only,
   per `Axiom.SDD/AGENTS.md`'s explicit-bootstrap-limits rule ("If
   something appears useful but belongs to the future ... document it
   ... and do not implement it now").
4. **Finding 3 and Finding 4** (the `configure.ts`/`sync.ts`
   inconsistency and the `CLAUDE.md`/`AGENTS.md` registry drift) are
   real but unrelated defects — worth their own small follow-up
   increments, not bundled into this one.

## Sequencing recommendation (revises parent roadmap)

The parent roadmap's INC-05 subagent sequence was:
migration-engineer → docs-skills-writer → adapter-engineer →
validator-reviewer. Given this audit's finding that **no adapter
content needs to change**, the sequence collapses:

- **docs-skills-writer is not needed** — there is no new canonical
  template content to write; INC-03's docs-skills-writer step already
  delivered it, and this audit found no gap in it that a
  docs-skills-writer would need to close.
- **adapter-engineer is not needed either**, contrary to what the task
  brief suggested confirming/revising — there is no wiring work to do
  because there is no divergent content to wire together. Recommending
  "adapter-engineer next" (as the task brief hypothesized) would be
  scope invention: it would mean asking adapter-engineer to build a
  dependency from adapters onto canonical `AGENTS.md` that does not
  currently need to exist, per Finding 2.
- **validator-reviewer is optional**, not mandatory: `TC-009` already
  covers adapter package-shape enforcement correctly for what exists
  today; there is no new consistency invariant for it to enforce until
  a future increment actually creates adapter content that overlaps
  with canonical `AGENTS.md`.

**Revised recommendation: close this chain of INC-05 here, with no
further subagent needed**, unless the user wants to pursue the
optional one-line cross-reference improvement (Recommendation item 3)
or the two adjacent defects (Finding 3, Finding 4) as separate,
explicitly-scoped follow-up increments.

## Open questions

- Does the user want the optional one-line cross-reference from
  `opencode`/`claude-code`/`cursor`'s `AGENTS.md` outputs back to the
  canonical repo-root `AGENTS.md` (Recommendation item 3)? Not
  blocking closure of this audit either way.
- Should Finding 3 (`configure.ts` mixing old/new copilot pipelines)
  and Finding 4 (`CLAUDE.md`/`AGENTS.md` registry drift) become their
  own tracked increments? Recommended, but not decided here — these
  are pre-existing defects unrelated to the `AGENTS.md`-adapter
  reconciliation question this increment was scoped to answer.

## Assumptions

- "Adapter genuinely derives from canonical AGENTS.md" was interpreted
  as: the adapter's generated content textually or structurally
  restates governance/structural-rule content that canonical
  `AGENTS.md` already owns. Under that definition, capability/skill
  listings (which canonical `AGENTS.md` does not render at all) do not
  count as overlap, even though three adapters happen to also produce
  a file literally named `AGENTS.md`.
- The `install-profile.json` / `ResolvedInstallProfile` data source is
  treated as an intentional, stable input for adapters (capability
  enablement, adapter target selection) — not itself a candidate for
  replacement by canonical `AGENTS.md`'s identity fields
  (`projectName`/`role`/`layout`), which serve a different purpose.

## Implementation notes

Audit only. No files were changed under `Axiom/packages/`,
`Axiom/apps/`, or `Axiom.SDD/`.

### Files read in full

- `Axiom/packages/document-bootstrap/src/{renderer,variables,writer,canonical-agents-md,index,idempotency,types}.ts`
- `Axiom/packages/adapters/README.md`
- `Axiom/packages/adapters/opencode/src/{agents-md,generator,types}.ts`
- `Axiom/packages/adapters/claude-code/src/{agents-md,generator,types}.ts`
- `Axiom/packages/adapters/github-copilot/src/{instructions,generator,types}.ts`
- `Axiom/packages/adapters/vscode/src/{settings,generator,types}.ts`
- `Axiom/packages/adapters/cursor/src/{settings,generator,types}.ts`
- `Axiom/packages/adapters/litellm/src/{config,generator,types}.ts`
- `Axiom/packages/installer/src/registry.ts`
- `Axiom/apps/cli/src/commands/sync.ts` (adapter dispatch section,
  `materializeAdapterOutputs`)
- `Axiom/apps/cli/src/commands/configure.ts` (adapter invocation
  section, grep-scoped to the relevant lines)
- `Axiom/packages/doctor/src/checks.ts` (`TC-009` /
  `runAdapterRuntimeCoverageCheck` section)
- `Axiom.Spec/specs/increments/INC-20260702-installer-wizard-reconcile-agents-md/README.md`
  (full)
- `Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/README.md`
  (full, for INC-05's original framing)

## Validation

`No validation command was found. Performed best-effort validation by inspecting changed files and checking consistency against the requested behavior.`

Note: this increment made no code changes, so there is nothing to
build/test/typecheck. Best-effort validation performed: every claim in
the "Findings" section above is sourced from directly reading the
listed files in full (not inferred from package names or prior
increment summaries), and cross-checked against
`Axiom/packages/installer/src/registry.ts`'s `GENERATED_FILES_BY_TARGET`
and `Axiom/apps/cli/src/commands/sync.ts`'s actual dispatch switch to
confirm the generated-file paths and data sources stated in the diff
table are accurate as of the current repository state.

## Result

Audited all 6 adapter packages, the document-bootstrap package (both
its old `ResolvedVariables`-based pipeline and its new canonical-
`AGENTS.md` pipeline), the installer's static generated-files registry,
the two invocation call sites (`sync.ts`, `configure.ts`), and the
`TC-009` doctor check. Found that **zero of the 6 adapters currently
duplicate or diverge from canonical `AGENTS.md`'s content** — they
occupy a disjoint content domain (capability/skill routing and tool
config, read from `ResolvedInstallProfile`/`install-profile.json`),
while canonical `AGENTS.md` owns governance/structural-rule content and
project identity, read from a separate, minimal
`CanonicalAgentsMdIdentity` input. This overturns the parent roadmap's
"potentially breaking, needs a shim" framing for INC-05: there is no
divergent instruction content to reconcile, so no shim, no adapter
wiring change, and no new canonical-template work is needed. Two
unrelated pre-existing defects were flagged as adjacent findings
(`configure.ts` mixing old/new copilot pipelines; a
`CLAUDE.md`/`AGENTS.md` naming drift between the installer registry and
the actual claude-code generator output) for separate follow-up, out of
this increment's scope.

## General spec integration

No integration into a `general-spec.md` was performed — it does not
exist in this repo (confirmed absent, consistent with prior increments
in this same chain). The one piece of knowledge worth flagging for
whoever eventually creates `general-spec.md`: **canonical `AGENTS.md`
and the 6 tool adapters are intentionally disjoint pipelines as of this
audit** — canonical `AGENTS.md` owns governance/identity content
(`@axiom/document-bootstrap`'s `canonical-agents-md.ts`), while
adapters own capability/skill/tool-config content
(`@axiom/adapters-*`, keyed on `ResolvedInstallProfile`). Any future
work that wants to merge these must treat it as new scope (adding
adapter content that doesn't exist today), not as fixing a divergence
(none exists today).

## Next step recommendation

**Close this audit chain here.** Per the "Sequencing recommendation"
section above, neither `docs-skills-writer` nor `adapter-engineer` has
concrete work to do: no adapter currently diverges from or needs to
derive from canonical `AGENTS.md`. If the user wants to pursue any of
the three optional items surfaced by this audit, open them as their
own small, explicitly-scoped increments rather than continuing INC-05's
original sequence:

1. One-line cross-reference from `opencode`/`claude-code`/`cursor`'s
   `AGENTS.md` outputs to the canonical repo-root `AGENTS.md`
   (cosmetic, low-risk, genuinely optional).
2. Reconcile `configure.ts`'s mixed old/new copilot-pipeline usage
   with `sync.ts`'s uniform new-pipeline usage (Finding 3).
3. Fix the `GENERATED_FILES_BY_TARGET` registry's stale
   `.claude/CLAUDE.md` entry to match the actual
   `.claude/AGENTS.md` output (Finding 4).

None of these are blocking, and none were requested as part of this
increment's scope.
