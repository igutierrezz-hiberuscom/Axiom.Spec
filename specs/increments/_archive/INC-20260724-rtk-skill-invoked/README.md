# Increment: RTK skill-invoked (sin hooks) + skill `terminal-output-efficient`

Status: closed
Date: 2026-07-24

## Goal

Make RTK (the optional terminal-output reducer already declared in the
toolchain catalog, `kind: input-optimizer`) usable **ONLY** via a skill —
never a git hook, never a global/transparent wrapper around commands — with
a clear when-to-use / when-NOT-to-use policy, an explicit fallback flow, and
a hard guard that critical content (Engram memory, the spec, ADRs,
compliance/security evidence) is never compressed by it.

This is **INC-T2** of Cluster T in the Axiom evolution plan
(`giggly-imagining-moore.md`, "Cluster T → INC-T2"): independent of Cluster
A and of INC-T1 (`cmm` replaces `graphify`/`codegraph`, closed). User's
exact intent, quoted in the brief: "RTK se instala pero sin los hooks; su
uso se incluye en las skills para que lo usen los agentes, y las skills
definen cuándo usarlo." Explicitly user-approved; not down-scoped.

## Context

Before this increment, RTK (`id: rtk`, `kind: input-optimizer`) was
catalog-only, with zero runtime anywhere in Axiom:

- `axiom.config/toolchain-catalog.yaml` declares it (`kind:
  input-optimizer`, `mvp: false`) alongside `serena`/`cmm`/`engram`/
  `context7`/`caveman`/`autoskills`.
- `packages/toolchain/src/probe.ts`'s `resolveProbeCommand` already
  returns `null` for `rtk` (grouped with `context7`/`caveman`/
  `autoskills`) — an honest "no known local version-probe contract",
  never a guess. The generic, pre-existing `detectToolState`/`axiom
  toolchain add --id rtk` machinery (`apps/cli/src/commands/
  toolchain.ts`'s `deriveKindForId`/`defaultToolEntry`) already derives
  `kind: input-optimizer` correctly and a default, passive,
  marker-based `detectionPaths` entry
  (`.axiom-state/${projectName}/toolchain/rtk/`) for it — this predates
  the increment and required no change.
- No git hook, no wrapper, no interceptor anywhere in the repo ever
  referenced RTK (confirmed by repo-wide search): the only pre-existing
  "no git hook" wiring found belongs to `cmm`'s freshness check
  (`packages/providers/src/code-intel/cmm-freshness.ts`) and
  `workspace-code-intel.ts`, both unrelated to RTK.
- No skill documented RTK's usage policy. The closest existing skill,
  `axiom-code-intelligence.md`, only namedrops "RTK" once in passing as
  an optional P1 tool, with no decision guidance.

Because RTK had no runtime to begin with, this increment is purely
additive: author the policy skill, register it in the skills catalog with
a verified `bundleHash`, and update the catalog's guard tests in
lockstep — the same `bundleHash` gotcha INC-T1 already handled
successfully for its own touched skills.

## Scope

- **New skill `axiom-terminal-output-efficient`** (id follows the existing
  `axiom-*` convention used by all 18 prior catalog entries — the
  un-prefixed `terminal-output-efficient` alternative offered in the brief
  was rejected as inconsistent with every other id in the catalog).
  Authored at `axiom.spec/target-axiom-skills/
  axiom-terminal-output-efficient.md`, mirroring the dominant structure of
  the other capability/tool-usage skills (`# Title` / `## Qué hace` /
  `## Cuándo se usa` / `## Cómo lo hace` with `###` sub-sections / `##
  Reglas absolutas` / `## Relación con otras piezas` / `## Estado`) —
  17 of 18 existing sources use this plain-heading structure; only
  `axiom-context-persistence.md` has a one-off YAML-frontmatter variant
  that no other skill (including its closest analogues,
  `axiom-code-intelligence`/`axiom-token-optimization`) follows, so it
  was not replicated (see Assumptions).
- **Skill body** encodes: a markdown decision table (RTK yes / RTK no,
  covering large-repetitive output, exploratory commands,
  errors-only-matter, known/stable tests, recoverable output on the
  "yes" side; unknown-error debugging, exact-event-order, full
  stack-trace, intermittent-failure investigation, full-evidence
  requirement, explicit human request on the "no" side); a fallback flow
  (run optimized → if it doesn't explain the failure, re-run WITHOUT RTK
  → save full output as an artifact when the case needs review → degrade
  to full output if RTK isn't installed, never blocking); a
  never-compress exclusion list (Engram memory, the spec and its
  increments/bugs, ADRs/decisions, compliance/security evidence); and an
  explicit "Reglas absolutas" prohibition against invoking RTK from a git
  hook or a global/transparent wrapper.
- **Catalog registration**: `axiom.config/skills-catalog.yaml` gained one
  entry (`axiom-terminal-output-efficient`, `version: 0.1.0`, `status:
  approved`, `securityCheckStatus: ok`, `bundleHash` = sha256 of the raw
  source bytes, computed with Node and verified to match what doctor's
  `TC-010` recomputes at read time).
- **Guard-test lockstep** (the bundleHash gotcha, same pattern INC-T1
  used): `packages/skills/tests/catalog.test.ts` (the real-catalog
  integration test: 18 → 19 entries, sorted id list updated, comment
  updated) and `packages/doctor/tests/skills.test.ts` (the real-repo
  `TC-010` smoke test: description and `/18\/18/` → `/19\/19/` regex
  updated). A brand-new, dedicated test block (`Scenario 5`) was added to
  `catalog.test.ts` asserting the new entry's metadata (name, version,
  source, status, securityCheckStatus) and a bundleHash that is
  RE-COMPUTED from the live source file in the test itself (not
  hardcoded), plus a second test asserting the source body actually
  contains the decision table, fallback flow, no-compression exclusion
  terms, and the hook/wrapper prohibition — so the policy content
  itself is guarded, not just its hash.
- **Toolchain probe (`packages/toolchain/src/probe.ts`)**: reviewed, NOT
  functionally changed. A documentation-only comment was added to
  `resolveProbeCommand` recording that this increment specifically
  reviewed the `rtk` branch and confirmed the existing honest `null`
  (no active version-probe) is correct — see Non-goals/Assumptions for
  why no new probe logic was added.
- **No agents-catalog change**: confirmed (`axiom.config/
  agents-catalog.yaml`) that skills and agents are NOT 1:1 in this
  catalog — only the 8 "process-flow" skills (spec-author, role-planner,
  role-implementer, sdd-orchestrator, phase-reviewer, spec-integrator,
  tech-context, qa-validator) have a matching agent wrapper; the other
  10 capability/discipline skills (including the 2 closest analogues to
  this one, `axiom-code-intelligence`/`axiom-token-optimization`) do
  not. No agent was added for the new skill, consistent with precedent.
- **No `workspace-process-surfaces.ts` TS-constant**: confirmed that only
  the 8 "process-flow" skills materialized per-repo-role during
  scaffold/adopt get a condensed TS-constant body in that file (because a
  scaffolded/adopted repo has no on-disk access to
  `axiom.spec/target-axiom-skills/`); the other 10 catalog-only skills
  (including the 2 closest analogues) are absent from that file and rely
  on `@axiom/skills`'s `materializeSkillSet`/`materializeSingleSkill`
  reading the source `.md` directly. The new skill follows the SAME
  catalog-only pattern — no TS-constant representation was added.

## Non-goals

- `cmm`/providers (INC-T1, closed) — untouched.
- The skills-wide concision policy (Caveman principles as skill-writing
  policy, INC-T3) — untouched; the new skill only carries a one-line,
  non-implementing cross-link noting it is independent of that future
  policy (which governs how skills are WRITTEN, not when terminal output
  is compressed).
- AutoSkills lock hygiene (INC-S1), worktrees (Cluster W), the unified
  `<project>.axiom` MCP (Cluster A, done) — untouched.
- A real, active version-probe for RTK (e.g. a fabricated `rtk --version`
  spawn). RTK's actual local CLI contract (if any) is not verified in
  this environment; `serena`/`cmm`/`engram` each got a real probe branch
  only because their binary/CLI shape was independently confirmed or
  documented elsewhere in the codebase (or, for `cmm`, deliberately
  flagged as an ASSUMPTION with full override support). Inventing one for
  `rtk` would violate `probe.ts`'s own "never a guess" rule. The
  pre-existing, generic, PASSIVE marker-based detection (`detectToolState`
  + `axiom toolchain add/show/validate`, unchanged, already produces
  `declared`/`absent`/`marker` honestly for `rtk`) already satisfies "so
  `axiom doctor`/`toolchain` know RTK is installed" without guessing —
  this is the "passive detection only" the brief allowed, already present,
  requiring no code change.
- A code-level (runtime) enforcement mechanism for the no-compression
  guard. See Assumptions for why the skill-level "Reglas absolutas"
  prohibition was chosen instead of a new doctor check.
- Migrating Axiom's own repos (`Axiom.SDD`/`Axiom.Spec`) — out of scope
  (deferred to the user per the plan).

## Acceptance criteria

- [x] A new skill (`axiom-terminal-output-efficient`) exists, authored at
      `axiom.spec/target-axiom-skills/axiom-terminal-output-efficient.md`,
      mirroring the existing skill sources' dominant structure.
- [x] The skill is registered in `axiom.config/skills-catalog.yaml` with a
      `bundleHash` that is the real sha256 of the source's raw bytes,
      verified to match what doctor's `TC-010` independently recomputes.
- [x] The skill body encodes: a when-to-use/when-NOT-to-use table (large/
      repetitive output, exploratory command, only-errors/summary-matter,
      known/stable tests, recoverable output vs. unknown-error debugging,
      exact-event-order, full-stack-trace, intermittent-failure
      investigation, full-evidence-required, explicit human request); a
      fallback flow (optimized → full re-run when it doesn't explain the
      failure → save full output as an artifact when the case needs
      review); and explicit exclusions (Engram, spec, ADRs, compliance/
      security evidence never compressed).
- [x] RTK is invoked ONLY from this skill: no git hook, no global/
      transparent wrapper exists anywhere in the repo (confirmed by
      repo-wide search both before and after the change).
- [x] Any toolchain support added is passive-detection-only, never an
      auto-piping interceptor. Concretely: no new probe logic was added
      (documented decision); the pre-existing passive marker-detection
      was reviewed and confirmed sufficient, with a comment recording the
      review.
- [x] A no-compression guard exists for Engram/spec/ADR/evidence. Chosen
      form: skill-level "Reglas absolutas" prohibition (no natural
      code/doctor chokepoint exists, since RTK has no runtime/wrapper
      code in Axiom to attach a runtime guard to) — documented explicitly
      in Assumptions, plus a doctor-independent unit test asserting the
      source text actually contains the exclusion terms.
- [x] `packages/skills/tests/catalog.test.ts` and
      `packages/doctor/tests/skills.test.ts` (the two guard tests hard-
      asserting the real catalog's skill count/ids) are updated to 19 in
      lockstep with the new entry.
- [x] A new, dedicated unit test asserts the new skill is in the catalog
      with a valid bundleHash and the expected metadata (name, version,
      source, status, securityCheckStatus).
- [x] `npm run build` (`tsc -b`) passes.
- [x] Targeted `vitest run` for `packages/skills`, `packages/doctor`,
      `packages/toolchain`, and the relevant `apps/cli/tests` skills/
      catalog/toolchain files passes, 0 new failures.
- [x] Full-repo `vitest run` passes with the exact expected delta (INC-T1's
      documented baseline of 297 files / 2990 tests, plus exactly the 2
      new tests this increment added: 297 files / 2992 tests, 0
      failures) — confirms zero regressions anywhere else in the repo.

## Open questions

None blocking.

## Assumptions

Each is a deliberate, narrowest-change resolution of an ambiguity the
brief left open:

1. Skill id: `axiom-terminal-output-efficient` (not the bare
   `terminal-output-efficient` alternative the brief also offered) —
   matches the id convention of all 18 pre-existing catalog entries
   without exception.
2. Skill body structure: plain headings (`# Title` / `## Qué hace` / …),
   NOT the one-off YAML-frontmatter variant used only by
   `axiom-context-persistence.md`. 17 of 18 existing sources — including
   both closest analogues, `axiom-code-intelligence` and
   `axiom-token-optimization` — use the plain-heading form; the
   catalog's own `CatalogEntry`/`bundleHash` mechanism (in
   `axiom.config/skills-catalog.yaml`) is what actually carries the
   "frontmatter-equivalent" metadata (id/name/version/source/status/
   bundleHash), making a second, in-body YAML frontmatter redundant and
   inconsistent with the dominant pattern.
3. No `workspace-process-surfaces.ts` TS-constant body: that file only
   condenses the 8 role-materialized "process-flow" skills (verified:
   its `SURFACES` record has exactly 8 entries, and grepping the whole
   `apps/cli/src`/`packages` tree for the other 10 catalog-only skill
   ids — including this new one's 2 closest analogues — found zero
   references outside tests). The new skill follows the SAME
   catalog-only materialization path (`@axiom/skills`'s
   `materializeSkillSet`, reading the source `.md` directly) as those 10.
4. No active RTK probe added to `packages/toolchain/src/probe.ts`: RTK's
   real local CLI/binary contract is unverified in this environment;
   `resolveProbeCommand` already documents (pre-existing comment) that
   `rtk` intentionally has no known contract, "typically hosted/remote
   or instruction-driven". Adding a guessed `rtk --version`-style probe
   would silently contradict that module's own "never a guess" honesty
   rule. A documentation-only comment was added instead, recording that
   this increment reviewed the branch and confirmed it correct — no
   functional/logic change, so no test risk.
5. No-compression guard implemented as a skill-level "Reglas absolutas"
   prohibition, not a new doctor/code check: RTK has zero runtime or
   wrapper code anywhere in Axiom (confirmed catalog-only, this
   increment and INC-T1's context both independently verified this) —
   there is no function that "runs a command, maybe through RTK" to
   attach a guard to. Inventing a doctor check that pattern-matches the
   skill's prose for the exclusion keywords would be a brittle,
   speculative addition (the bootstrap guardrails explicitly discourage
   "speculative architecture" and ask to "keep changes small and
   focused") for a prohibition that has no code chokepoint to guard in
   the first place. This mirrors the EXISTING precedent in this very
   catalog: none of the other discipline skills' "Reglas absolutas"
   (e.g. `axiom-structured-doubts`, `axiom-plan-drift-alignment`) are
   enforced by doctor/code either — the skill text IS the enforceable
   contract for this class of rule throughout the catalog. A dedicated
   unit test (not a doctor check) was added instead, asserting the
   policy text is actually present in the source, so silent prose drift
   is still caught by the standard test suite.
6. Cross-link to INC-T3 (skill-writing concision policy) kept to a single
   sentence, explicitly scoped as "independent of" rather than
   depending on or implementing it — honors the brief's "a small
   cross-link is fine" allowance without reaching into INC-T3's scope.

## Implementation notes

All files changed live in `Axiom/` (the product monorepo) unless noted.

- New: `axiom.spec/target-axiom-skills/axiom-terminal-output-efficient.md`.
- Changed: `axiom.config/skills-catalog.yaml` (new entry + explanatory
  comment), `packages/skills/tests/catalog.test.ts` (18→19 + new
  `Scenario 5` block, 2 new tests), `packages/doctor/tests/skills.test.ts`
  (18→19 description + regex), `packages/toolchain/src/probe.ts`
  (comment-only, no logic change).
- Explicitly NOT changed (see Scope/Assumptions for why):
  `axiom.config/agents-catalog.yaml`, `apps/cli/src/commands/
  workspace-process-surfaces.ts`, any `packages/capability-model`/
  `packages/providers` file, `axiom.config/toolchain-catalog.yaml` (the
  `rtk` entry there already existed, unchanged), any git hook
  configuration (none existed to begin with).

## Validation

Ran from `C:\repos\Axiom Workspace\Axiom`:

- `npm run build` (`tsc -b`) — PASSED, clean, exit 0, no output.
- Targeted `npx vitest run packages/skills` — **6 files, 73 tests
  passed** (up from 71 pre-existing; the 2 new `Scenario 5` tests).
- Targeted `npx vitest run packages/doctor` — **18 files, 182 tests
  passed**, including the updated `TC-010` smoke test now asserting
  `19/19 entries consistentes`.
- Targeted `npx vitest run packages/toolchain` — **3 files, 29 tests
  passed** (comment-only `probe.ts` change; no behavior/test change).
- Targeted `npx vitest run` on `apps/cli/tests/{skills,
  workspace-skills,workspace-autoskills,workspace-process-surfaces,
  toolchain}.test.ts` — **5 files, 65 tests passed**.
- Full-suite `npx vitest run` (whole repo) — **297 files, 2992 tests
  passed. Zero failures.** Matches INC-T1's documented green baseline
  (297 files / 2990 tests) plus exactly the 2 new tests added here —
  confirms no regressions anywhere else in the repo.

No pre-existing failures were observed to classify — the suite was fully
green before and after in every scope run.

## Result

Implemented. RTK remains entirely skill-invoked: the new
`axiom-terminal-output-efficient` skill is the ONLY surface in Axiom that
documents when/how to use it, with a concrete decision table, a fallback
flow (optimized → full re-run → save-as-artifact → degrade-gracefully-if-
absent), and an explicit, hard-coded exclusion list for Engram/spec/ADR/
compliance evidence. No hook, wrapper, or interceptor was added or existed.
The toolchain's existing passive, marker-based detection for `rtk` was
reviewed and left correct (documented, not changed); no active/guessed
probe was fabricated. The catalog's `bundleHash` gotcha was honored: the
new entry's hash was computed from the real source bytes and both
hardcoded-count guard tests (`packages/skills/tests/catalog.test.ts`,
`packages/doctor/tests/skills.test.ts`) were updated in lockstep, plus a
new dedicated test guarding both the entry's metadata and the policy
content itself. Full build + full test suite are green (297/297 files,
2992/2992 tests).

## General spec integration

Integrado en la spec canónica al cierre del batch INC-20260724-* (integración
única de toda la tanda). Ficheros que recibieron el conocimiento de este
incremento:

- `06_Integraciones_y_Capacidades.md` — RTK invocado solo por la skill
  `axiom-terminal-output-efficient` (sin hooks/wrapper), tabla de decisión +
  fallback (catálogo 18→19).
- `07_Gobierno_y_Seguridad.md` — exclusiones never-compress
  (Engram/spec/ADR/evidencia) y prohibición de hook/wrapper.
- `08_Glosario.md` — `axiom-terminal-output-efficient` (skill).
- `01_Requisitos_Funcionales.md` — RF-AXM-045.
