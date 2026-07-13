# INC-20260713-ns1-process-skills

NS-1 — author the SDD **process (HOW)** skills/agents for the 3 operating flows.
This is content + catalog authoring: it adds reusable, role/repo-parameterizable
process documents to the official Axiom skills/agents catalogs so that the sdd/spec
MCP can serve them and a later increment (NS-2) can generate them into repos. The
build is GREEN with the existing test suite and MUST stay green; every change is
additive.

## Goal

Give Axiom the reusable **process content** that drives its 3 operating flows via
MCP, generalized from KVP25's proven agents and re-expressed Axiom-native around the
real MCP tools:

1. **Analyst** (spec repo) — create/expand an increment or bug: discover before
   writing, close ambiguities, guarantee functional coverage, author safe (draft)
   spec content.
2. **Architect** (spec repo) — plan: read spec + technical context, then emit **one
   plan per registered role** with target repos and an allowed write scope, using
   code-intel with a physical-path fallback.
3. **Implementer** (a role's code repo) — read the WHAT bundle
   (`spec.implementationContextRead`), load repo-local skills, use code-intel with
   fallback, stay within the allowed write scope, validate, and drive confirm-gated
   git.

## Scope — author process skills + agents into the catalog

Three rich process **skills** (the HOW the MCP serves) under
`Axiom/axiom.spec/target-axiom-skills/`, each registered in
`axiom.config/skills-catalog.yaml` with a correct sha256 `bundleHash`:

- **`axiom-spec-author`** (analyst flow) — reads `sdd.skillIndexRead`,
  `spec.incrementRead` / `spec.bugRead`, `spec.technicalContextIndexRead` /
  `spec.recommendedContextList`, `spec.skillCatalogRead` / `spec.skillRead`; authors
  draft spec content; moves state via confirm-gated `sdd.transitionApply`.
- **`axiom-role-planner`** (architect flow) — reads `sdd.projectReposRead`,
  `spec.incrementRead`, `spec.technicalContextIndexRead` /
  `spec.recommendedContextList`, `spec.planRead`, `spec.adrIndexRead` /
  `spec.decisionIndexRead`; gathers code facts via code-intel MCP (serena/codegraph)
  with a `{repoPath}` fallback; emits one role plan per registered role with
  `targetRepos` + `allowedWriteScope`; moves state via `sdd.transitionApply`.
- **`axiom-role-implementer`** (implementer flow) — reads
  `spec.implementationContextRead` (the WHAT bundle), `sdd.skillIndexRead`,
  `spec.skillCatalogRead` / `spec.skillRead`, `sdd.allowedWriteScopeRead`; uses
  code-intel with fallback; validates with `sdd.changesValidate`; drives
  confirm-gated `sdd.gitRoleBranch` / `sdd.gitCommitSync` (no push by default) and
  `sdd.transitionApply`.

Three thin materializable **agent** contracts (the invocable per-role wrappers NS-2
generates) under `Axiom/axiom.spec/target-axiom-agents/`, registered in
`axiom.config/agents-catalog.yaml` with correct sha256 `bundleHashes`: same three
ids, each pointing at its skill as `relatedSkills`, roles mapped onto the closed
`AgentRole` union (`axiom-spec-author` = `planning`, `axiom-role-planner` =
`planning`, `axiom-role-implementer` = `sdd-orchestration`).

All content is role/repo-parameterizable with literal placeholders the NS-2
generator fills: `{role}`, `{specRepo}`, `{repoPath}`, `{projectId}`. It
cross-references the existing platform meta-skills (`axiom-sdd-orchestrator`,
`axiom-capability-router`, `axiom-code-intelligence`, `axiom-context-persistence`,
`axiom-token-optimization`) instead of duplicating them.

## Non-goals

- **No generator changes (NS-2).** The adapter generators and `renderAgentsMd` are
  untouched. NS-1 only authors catalog content; NS-2 generates it into repos,
  substituting placeholders.
- **No code-intel MCP wiring (NS-3).** Code-intel-with-fallback is only *referenced*
  in the prose; no serena/codegraph server is wired here.
- **No new AgentRole and no new workflow/command.** Roles are mapped onto the
  existing closed union.
- **No 4th flow.** A `spec-reviewer` / integrator was deliberately left out to keep
  scope sane — review is already covered by `axiom-reviewer`,
  `axiom-product-reviewer`, and `axiom-security-reviewer`.

## Acceptance

- [x] 3 skill sources authored under `axiom.spec/target-axiom-skills/` and registered
      in `skills-catalog.yaml` with correct sha256 bundleHashes.
- [x] 3 agent sources authored under `axiom.spec/target-axiom-agents/` and registered
      in `agents-catalog.yaml` with correct sha256 bundleHashes.
- [x] Each doc explicitly references the MCP tools its flow calls and uses the
      `{role}` / `{specRepo}` / `{repoPath}` / `{projectId}` placeholders.
- [x] Doctor **TC-010** (skills catalog) and **TC-011** (agents catalog) stay green
      with the new entries (source exists + bundleHash matches).
- [x] `npm run build`, `npm test`, `npm run typecheck`, `npm run doctor` all green;
      no test weakened (count/membership assertions updated to reflect the expanded
      catalogs).

## Decisions

See `metadata.yml` (`decisions:` D-001..D-003, `openQuestions:` Q-001). The
skills-and-agents choice is Q-001/D-001; the role mapping is D-002; the test
count/membership update is D-003.

## Result

**Archived — complete and green.** NS-1 shipped as pure additive catalog content; no
generator or runtime code touched.

### New process skills (rich HOW) — `axiom.spec/target-axiom-skills/`, in `skills-catalog.yaml`

- **`axiom-spec-author`** (analyst) — MCP: `sdd.skillIndexRead`,
  `sdd.transitionApply`, `spec.incrementRead`, `spec.bugRead`,
  `spec.technicalContextIndexRead`, `spec.recommendedContextList`,
  `spec.skillCatalogRead`, `spec.skillRead`, `memory.contextRecall`. Phases:
  anchor+context → discovery → ambiguity checklist → draft spec → functional coverage
  (`CF-xx`) → versioning/state.
- **`axiom-role-planner`** (architect) — MCP: `sdd.skillIndexRead`,
  `sdd.projectReposRead`, `sdd.transitionApply`, `spec.incrementRead`,
  `spec.planRead`, `spec.technicalContextIndexRead`, `spec.recommendedContextList`,
  `spec.adrIndexRead`, `spec.decisionIndexRead`, `memory.decisionRecall`, plus
  code-intel (serena/codegraph) with `{repoPath}` fallback. Emits one role plan per
  registered role with `targetRepos` + `allowedWriteScope`.
- **`axiom-role-implementer`** (implementer) — MCP: `spec.implementationContextRead`
  (the WHAT bundle), `spec.skillCatalogRead`, `spec.skillRead`, `sdd.skillIndexRead`,
  `sdd.allowedWriteScopeRead`, `sdd.changesValidate`, `sdd.gitRoleBranch` /
  `sdd.gitCommitSync` (confirm-gated, no-push), `sdd.transitionApply`,
  `memory.decisionRecall`, plus code-intel with `{repoPath}` fallback. Stays within
  write scope.

### New agent contracts (thin) — `axiom.spec/target-axiom-agents/`, in `agents-catalog.yaml`

- `axiom-spec-author` (`role: planning`), `axiom-role-planner` (`role: planning`),
  `axiom-role-implementer` (`role: sdd-orchestration`). Each wraps its matching skill
  via `relatedSkills` and lists the same MCP tools.

All content is parameterized with literal placeholders `{role}`, `{specRepo}`,
`{repoPath}`, `{projectId}` (NS-2 substitutes them; literal at cataloging time so
bundleHashes stay stable). Cross-references the existing meta-skills
(`axiom-sdd-orchestrator`, `axiom-capability-router`, `axiom-code-intelligence`,
`axiom-context-persistence`, `axiom-token-optimization`) instead of duplicating them.

### bundleHash integrity

Each source's `bundleHash` is `sha256:` of the exact file content
(`crypto.createHash('sha256')`), matching what doctor TC-010/TC-011 recompute. Doctor
TC-010 and TC-011 both PASS with the 10-entry catalogs.

### Gate

`npm run build` (`tsc -b`) ✓ · `npm run typecheck` ✓ · `npm run doctor` PASS (exit 0,
TC-010 + TC-011 green) · `npm test` **2754/2754** (272 files) ✓. No test weakened —
four real-catalog assertions were updated from 7→10 (doctor TC-010/TC-011 smoke +
`@axiom/skills` and `@axiom/agents` catalog integration counts, plus the agents id/role
membership lists) to reflect the intentionally expanded source of truth.

### Deliberately out of scope

NS-2 (generator/`renderAgentsMd` changes that materialize these into repos) and NS-3
(wiring the code-intel MCP servers) — code-intel-with-fallback is only referenced in
prose. No 4th flow (`spec-reviewer`/integrator) — review is already covered.
