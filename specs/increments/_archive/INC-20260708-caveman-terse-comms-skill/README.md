# Increment: Caveman-philosophy terse inter-agent comms skill (AB9)

Status: closed
Date: 2026-07-08

## Goal

Add a curated, LOCAL, bundled, opt-in-in-spirit skill (`axiom-terse-comms`)
that codifies caveman's token-compression PHILOSOPHY — but scoped strictly
to Axiom-agent-to-Axiom-agent communication (handoffs, subagent briefs,
structured status). Pair it with a short "Inter-agent communication style"
guidance block in the canonical `AGENTS.md`. Both artifacts must make the
human/agent split explicit and hard: anything a human reads or validates
(spec docs, increment READMEs, PR/commit text, user-facing summaries,
generated documentation) stays light and readable, never compressed.

## Context

- `C:\repos\caveman` is an external tool that rewrites agent OUTPUT into a
  terse "caveman" notation (symbols, fragments, dropped filler; ~65% output
  token reduction per its own benchmark table) via `/caveman` skill
  triggers. Axiom does not vendor or install this tool — only borrows the
  underlying idea (terser communication saves tokens) for one narrow
  surface: agent-to-agent traffic internal to Axiom's own SDD workflow.
- `apps/cli/src/commands/workspace-skills.ts` establishes the "bundle seed
  skill content as TS constants, materialize via the real `@axiom/skills`
  engine (`loadSkillRegistry` + `applySkillSet` + `computeSkillBundleHash`)"
  pattern already used for the 3 canonical Axiom skills
  (`axiom-sdd-orchestrator`, `axiom-context-persistence`,
  `axiom-capability-router`) plus the 2 `DEFAULT_DESIRED_SKILLS`
  (`serena-this-project`, `context7-this-project`). This increment adds a
  6th seed id to `CANONICAL_SEED_SKILLS`/`CANONICAL_SEED_SOURCES`, reusing
  the exact same catalog/materialization/role-index plumbing — no new
  mechanism.
- `packages/skills/src/materialize.ts`'s `renderSkillContent` prepends a
  minimal, stable `id`/`name`/`version` YAML frontmatter to whatever body
  Markdown a seed source provides; `Axiom.Spec/templates/product-skill-
  template.md`'s 18-field frontmatter contract is the FULL product-skill
  registry shape, not what the bundled seed catalog actually emits today —
  none of the 5 existing seed skills carry it either, so this increment
  does not introduce a new deviation.
- `packages/document-bootstrap/src/canonical-agents-md.ts`'s
  `renderCanonicalAgentsMd` already has the established "optional field on
  `CanonicalAgentsMdIdentity` -> conditional section, omitted when unset"
  pattern (`enabledCodeIntelProviders`, `enabledMemoryProvider`,
  `ruleScopes` from AB6/AB8). This increment adds one more such field
  (`interAgentTerseComms?: boolean`) and one more such section
  ("Inter-agent communication style"), following the same shape.
- Per the batch brief (AB8's closing note): this is AB9 in the sequential
  autopilot roadmap. AB8 (rules-layer) closed with "Ready for AB9 (caveman
  skill) — no blockers."

## Scope

- `apps/cli/src/commands/workspace-skills.ts`:
  - New seed id `axiom-terse-comms` added to `CANONICAL_SEED_SKILLS` (name:
    "Axiom Terse Comms", version `0.1.0`) and its body Markdown added to
    `CANONICAL_SEED_SOURCES`. Content (concise, curated):
    - States the terse convention applies ONLY to inter-agent traffic:
      agent-to-agent handoffs, subagent briefs, structured status passed
      between Axiom agents.
    - States the hard, explicit NEVER-compress list: spec docs (00-08),
      increment/bug READMEs, decision docs, PR/commit messages, user-facing
      summaries, generated documentation — these stay light and readable
      for a human, always.
    - A short symbol legend (subset, curated, not the full caveman
      notation) so terse inter-agent messages stay decodable: e.g. `→`
      (leads to / then), `∀` (for all / every), `!` (important / warning),
      `⊥` (blocked / failed), `?` (open question), `~` (about/approx),
      `ok`/`fail` (status).
    - Explicitly states this is guidance (opt-in in spirit), not a runtime
      compressor — no tool auto-rewrites anything.
  - `buildRoleIndex`: adds `axiom-terse-comms` to `available` (tagged
    `comms`), NOT `mandatory` (opt-in in spirit, per the brief).
  - `SEED_DESIRED_IDS`/`allSeedMeta` pick it up automatically (both are
    derived from `CANONICAL_SEED_SKILLS`).
- `packages/document-bootstrap/src/canonical-agents-md.ts`:
  - New optional field `CanonicalAgentsMdIdentity.interAgentTerseComms?:
    boolean`.
  - New `renderInterAgentCommsBlock` helper + "Inter-agent communication
    style" section, emitted only when `interAgentTerseComms === true`.
    States the same split as the skill (terse for agent-to-agent, light/
    readable for anything human-facing) plus the same short symbol legend,
    so an implementation agent following `AGENTS.md` alone (without having
    applied the skill) still gets the guidance.
  - No wiring into `runWorkspaceSetup`/`WorkspaceSetupSpec` in this
    increment (see Non-goals/Assumptions) — the field is rendered when a
    caller sets it, mirroring how `ruleScopes`/`enabledMemoryProvider` were
    introduced as pure-renderer capabilities before any wizard wiring
    existed for them, if ever.

## Non-goals

- No runtime output-compressor, no auto-rewriting of any agent output, no
  vendoring of `C:\repos\caveman`'s code or install script.
- No compression of ANY human-facing generation (specs, READMEs, commit/PR
  text, user summaries, generated docs) — explicitly and permanently out of
  scope, and the skill/AGENTS.md content says so.
- No new CLI surface, no new package, no new npm dependency.
- No wizard/`WorkspaceSetupSpec` wiring to auto-enable
  `interAgentTerseComms` — the brief scopes this increment to "opt-in in
  spirit (guidance, not enforced compression)"; a future increment can wire
  a wizard toggle if desired.
- No integration into `Axiom.Spec/specs/00-08` (reserved for the
  autopilot's final consolidation pass across the batch).
- No git commits, ever.

## Acceptance criteria

- [x] `axiom-terse-comms` is present in the bundled seed catalog
      (`CANONICAL_SEED_SKILLS`) and materializes via the existing
      `loadSkillRegistry`/`applySkillSet` engine like every other seed
      skill; its `SKILL.md` body contains the human-never-compress list and
      the symbol legend; `bundleHash` is computed via
      `computeSkillBundleHash` (not hardcoded); `loadSkillsCatalog` parses
      the catalog with the new id included.
- [x] `axiom-terse-comms` appears in each role's `skills-index` as
      `available` (not `mandatory`).
- [x] `renderCanonicalAgentsMd` renders an "Inter-agent communication
      style" section only when `interAgentTerseComms === true`, containing
      the terse-for-agents / light-for-humans split and the symbol legend;
      the section is absent when the field is unset or `false`.
- [x] Nothing in this increment auto-compresses any human-facing output —
      confirmed by inspection (no new runtime transform touches generated
      spec/README/commit content).
- [x] `npm run build` (`tsc -b`) is clean.
- [x] `npx vitest run apps/cli packages/skills packages/document-bootstrap`
      passes; full suite (`npx vitest run --no-file-parallelism`) shows 0
      regressions vs. the `2030/2030` baseline.

## Open questions

None blocking. Ambiguities resolved under Assumptions.

## Assumptions

- **Seed id and metadata**: `axiom-terse-comms`, name "Axiom Terse Comms",
  version `0.1.0` — follows the exact naming/versioning convention of the
  3 existing canonical seed skills.
- **`available`, not `mandatory`, in the role index**: the brief says
  "Opt-in in spirit (guidance, not enforced compression)". The two existing
  `mandatory` entries (`axiom-sdd-orchestrator`, `axiom-context-
  persistence`) are baseline SDD hygiene; terse inter-agent comms is a
  narrower, optional style preference, so it belongs in `available` next to
  `axiom-capability-router`/`serena-this-project`/`context7-this-project`.
- **`interAgentTerseComms` defaults unset/omitted, no wizard wiring**: the
  brief's "opt-in in spirit" plus the explicit non-goal list (no auto-
  compressor, no forced behavior) reads as "make the guidance available,
  do not turn it on by default anywhere yet." `renderCanonicalAgentsMd`
  gains the capability (pure, tested both ways) without
  `runWorkspaceSetup`/`WorkspaceSetupSpec` ever setting it `true` in this
  increment — consistent with how the renderer's existing optional fields
  (`enabledCodeIntelProviders`, `ruleScopes`) were each introduced as a
  pure capability first.
- **Symbol legend is a small curated subset, not caveman's full notation**:
  the brief asks for "a short symbol legend so terse messages remain
  decodable" — kept to ~6 symbols with unambiguous meanings
  (`→ ∀ ! ⊥ ? ~`) plus `ok`/`fail`, enough to demonstrate and constrain the
  convention without inventing a large bespoke symbol system.
- **Product-skill-template's 18-field frontmatter is not applied**: none of
  the 5 pre-existing bundled seed skills carry it (confirmed by reading
  `workspace-skills.ts`); `renderSkillContent` only ever emits
  `id`/`name`/`version`. Applying the full contract to only the 6th seed
  skill would be an inconsistent, unrequested scope increase.

## Implementation notes

Files edited:
- `Axiom/apps/cli/src/commands/workspace-skills.ts` — new seed id
  `axiom-terse-comms` (metadata + body in `CANONICAL_SEED_SOURCES`),
  `available` entry in `buildRoleIndex`.
- `Axiom/apps/cli/tests/workspace-skills.test.ts` — `SEED_IDS` extended
  with the new id (covers catalog/materialization/idempotency
  assertions that iterate `SEED_IDS`); new assertions for the SKILL.md
  body content (never-compress list + symbol legend) and for the
  role-index `available` placement.
- `Axiom/packages/document-bootstrap/src/canonical-agents-md.ts` — new
  optional `CanonicalAgentsMdIdentity.interAgentTerseComms` field,
  `renderInterAgentCommsBlock` helper, new conditional section.
- `Axiom/packages/document-bootstrap/tests/canonical-agents-md.test.ts` —
  new scenario for the "Inter-agent communication style" section (present
  when `true`, absent when unset/`false`).

## Validation

- `npm run build` (`tsc -b`, from `Axiom/`): clean, no errors.
- `npx vitest run apps/cli packages/skills packages/document-bootstrap`:
  target-scoped pass, 0 failures.
- `npx vitest run --no-file-parallelism` (full suite): 0 regressions vs.
  the `2030/2030` baseline (new tests added, none removed/weakened).

Exact tails recorded in the final response of this increment's execution.

## Result

Added `axiom-terse-comms` to Axiom's bundled seed skill catalog (available,
not mandatory) and an optional "Inter-agent communication style" section
to the canonical `AGENTS.md` renderer. Both artifacts encode the same
split: terse, symbol-legend-backed shorthand is acceptable ONLY for
Axiom-agent-to-Axiom-agent traffic (handoffs, subagent briefs, structured
status); anything a human reads or validates (specs, increment/bug docs,
decision docs, PR/commit messages, user summaries, generated
documentation) must stay light and readable, always. No runtime
compressor was built; no human-facing generation path was touched.

## General spec integration

Integrated into `Axiom.Spec/specs/00-08` by the batch's final
consolidation pass (autopilot orchestrator). Landed in:

- **06_Integraciones_y_Capacidades.md** — PRIMARY. New subsection "Skill
  de comunicación terse inter-agente (`axiom-terse-comms`)": the 6th
  bundled seed (available/opt-in), the matching optional AGENTS.md
  "Inter-agent communication style" block (`interAgentTerseComms`), and
  the hard human-vs-agent split (nothing human-facing is ever
  compressed; not a runtime compressor).
- **08_Glosario.md** — new term: `axiom-terse-comms`.

This is a documented (not enforced) convention, so no `01`/`02`/`03`
change was warranted — it adds no RF/NFR/data-model surface.
