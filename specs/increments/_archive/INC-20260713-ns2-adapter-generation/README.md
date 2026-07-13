# NS-2 — generate per-adapter, per-role invocable agents/commands/skills

## Goal

Turn the process content authored in NS-1 (`axiom-spec-author`, `axiom-role-planner`,
`axiom-role-implementer`) into REAL, adapter-native, **invocable** surfaces materialized
into each workspace repo, with the literal placeholders (`{role}`, `{specRepo}`,
`{repoPath}`, `{projectId}`) filled from the resolved topology. Each surface instructs the
AI, in the adapter's native shape, to call the sdd-MCP for the HOW, the spec-MCP for the
WHAT (especially `spec.implementationContextRead` for the implementer), use repo-local
skills, and use code-intel with a physical-path fallback (fallback text only; wiring the
actual code-intel MCP server is NS-3).

## Scope

1. **Placeholder rendering + role differentiation of AGENTS.md.** `renderAgentsMd`
   (claude-code + opencode) substitutes `{role}` / `{description}` / `{product.id}` /
   `{paths.specRoot}` / `{paths.sddRoot}` / `{discovery.modeOrder}` in the template
   preamble from the resolved topology/profile, so a spec repo's AGENTS.md differs from a
   code repo's. Opt-in via an `AgentsMdRenderContext` (absent = byte-identical legacy
   behavior). `generateWorkspaceAdapters` computes and passes a per-repo context.
2. **Per-role materialization of the NS-1 process surfaces** into each repo, by role:
   - spec repo (role `spec`): `axiom-spec-author` + `axiom-role-planner` (analyst + architect).
   - each code repo (role `code`): `axiom-role-implementer`, parameterized for that role's slug + repo.
   - sdd/control repo (role `sdd`): `axiom-sdd-orchestrator` (orchestrator/process surface).
   Wired into `runWorkspaceSetup` (so `runWorkspaceAdopt`, which delegates to it, and
   `repo add`/`role add` produce them too).
3. **Per-adapter shape.** claude-code + opencode = FULL native surfaces; cursor /
   github-copilot / copilot-vscode = rendered role-differentiated instruction file; every
   repo also gets adapter-agnostic portable surfaces under `.axiom/{agents,commands,skills}/`.
4. **skills-index wiring.** Register the materialized process skills in the repo's
   `skills-index/<role>.yaml` so `sdd.skillIndexRead` and
   `implementationContextRead.mandatory.repoSkills` return them, and reconcile-merge them
   into opencode `skills-lock.yaml` `installed` (no longer `[]`).

## Non-goals

- Wiring the actual code-intel MCP server (serena/codegraph) — that is **NS-3**. NS-2 only
  emits the physical-path fallback text inside the surfaces.
- Changing the NS-1 catalog entries or their `bundleHash` (placeholders stay literal at
  cataloging time).
- Removing any adapter target or changing the `runWorkspaceSetup` public signature.

## Per-adapter depth (decision)

| Adapter | Depth shipped |
|---------|---------------|
| claude-code | FULL — `.claude/agents/<id>.md` (frontmatter name/description + body) + `.claude/commands/<id>.md`; AGENTS.md placeholders filled |
| opencode | FULL — `.opencode/agents/<id>/SKILL.md` + `AGENT.md`; skill reconcile-merged into `.opencode/skills-lock.yaml installed`; AGENTS.md placeholders filled |
| cursor | Instruction file — `.cursor/rules/axiom-<id>.md` (role-differentiated, references surfaces + MCP) |
| github-copilot | Instruction file — `.github/instructions/axiom-<id>.instructions.md` |
| copilot-vscode | Instruction file — `.vscode/axiom-<id>.instructions.md` |
| all repos (any adapter) | Portable — `.axiom/agents/<id>.md`, `.axiom/commands/<id>.md`, `.axiom/skills/<id>/SKILL.md` |
| litellm / antigravity / visual-studio-2026 | Rely on the portable `.axiom/` surfaces (no dedicated native instruction shape) — deferred richness |

## Acceptance

- After `workspace setup`/`adopt` of a multi-repo project:
  - the spec repo has `axiom-spec-author` + `axiom-role-planner` surfaces;
  - each code repo has an `axiom-role-implementer` surface parameterized for its role;
  - AGENTS.md has NO raw `{placeholder}` and the spec repo's differs from a code repo's;
  - the code repo's `skills-index/<role>.yaml` / `implementationContextRead.repoSkills`
    includes `axiom-role-implementer`;
  - opencode `skills-lock.yaml installed` is non-empty (contains the process skills);
  - claude-code + opencode native outputs are present.
- The implementer surface references `spec.implementationContextRead`, repo-local skills,
  and the code-intel-with-physical-fallback text; placeholders are filled.
- Build / test / typecheck / doctor stay green.

## Result

Shipped, green on all gates (build / test / typecheck / doctor).

**1. Placeholder rendering + role differentiation of AGENTS.md.** `renderAgentsMd` in
both `packages/adapters/claude-code/src/agents-md.ts` and
`packages/adapters/opencode/src/agents-md.ts` now takes an optional
`AgentsMdRenderContext` (`{role,description,productId,specRoot,sddRoot,discoveryModeOrder}`)
and substitutes the preamble placeholders via `substitutePreamblePlaceholders`. Opt-in:
omitting it preserves the byte-identical legacy preamble, so no pre-existing generator
test broke. `generateWorkspaceAdapters` builds a per-repo context (from
`buildAgentsMdRepoContext` in `workspace-setup.ts`, deriving `discovery.modeOrder` from the
resolved profile) and passes it to the opencode/claude-code generators. New generator
scenarios (claude-code Scenario 6, opencode Scenario 7) assert both the substitution and
the opt-in literal-preserving behavior.

**2. Per-role materialization.** New module `apps/cli/src/commands/workspace-process-surfaces.ts`
bundles the NS-1 process surfaces as TS constants and materializes them by repo role
(spec → `axiom-spec-author` + `axiom-role-planner`; code → `axiom-role-implementer`
parameterized for the repo's functional role; sdd → `axiom-sdd-orchestrator`), filling
`{role}`/`{repoPath}`/`{specRepo}`/`{projectId}`. Wired into `runWorkspaceSetup` (step
7b-bis — so `runWorkspaceAdopt`, which delegates to it, produces them too) and into
`runRepoAdd` (new-repo block). Not gated on `created` (adopted repos get surfaces); skips
repos whose `axiom.yaml` belongs to another project.

**3. Per-adapter shape.** See the table above. claude-code + opencode FULL; cursor /
github-copilot / copilot-vscode = role-differentiated instruction file; every repo also
gets portable `.axiom/{agents,commands,skills}/` surfaces. litellm / antigravity /
visual-studio-2026 rely on the portable surfaces (deferred native richness — scope
realism for an L/XL increment).

**4. skills-index wiring.** `materializeProcessSurfaces` augments (or creates) the repo's
`skills-index/<role>.yaml` with the process skill as a mandatory entry, and reconcile-merges
it into opencode `skills-lock.yaml installed` (no longer `[]`). Verified e2e: a code repo's
`skills-index/backend.yaml` mandatory now includes `axiom-role-implementer` — which is what
`sdd.skillIndexRead` and `implementationContextRead.mandatory.repoSkills` (which reads the
target repo's index via `getSkillIndex`) return.

**e2e (scratch multi-repo KVP).** Spec repo (`Kvp.Spec`) got `axiom-spec-author` +
`axiom-role-planner` (`.claude/agents`, `.opencode/agents`, `.axiom/agents`); `Kvp.Api`
(role backend) got only `axiom-role-implementer` with `rol \`backend\``,
`.` repoPath, referencing `spec.implementationContextRead`, repo-local skills, and the
code-intel physical-path fallback, with no raw placeholders. AGENTS.md preambles differ
spec-vs-code and contain no raw `{placeholder}`. opencode `skills-lock.yaml installed`
contains `axiom-role-implementer`.

**Gate:** `npm run build` ✓ · `npm test` 2763 passed (0 real failures; 4 I/O/subprocess
files hit the known 5000ms load-timeout under heavy parallelism and pass in isolation) ·
`npm run typecheck` ✓ · `doctor` PASS (45/57 OK, 0 FALLO; TC-010/TC-011 catalog coverage
unchanged). Tests updated for behavior that legitimately changed: two workspace-setup
"no seed skills" assertions (now keyed on seed-only signals since process surfaces
legitimately populate `.opencode/agents`/`skills-index`), one adopt-conformance
`skills-index` assertion (`must-reconcile-by-hand` → `added`).

**Deferred (out of scope):** code-intel MCP server wiring (serena/codegraph) = NS-3;
native surface richness for litellm/antigravity/visual-studio-2026.
