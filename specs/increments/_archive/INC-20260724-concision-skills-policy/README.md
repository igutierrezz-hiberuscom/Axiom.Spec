# Increment: Concision principles (Caveman) as a shared skills policy

Status: closed
Date: 2026-07-24

## Goal

Adopt the *concision philosophy* of Caveman as a shared, reusable policy
that Axiom skills follow — WITHOUT installing or integrating Caveman as a
runtime dependency. User's intent, quoted in the brief: "de Caveman
incluimos sus principios de concisión dentro de las skills."

This is **INC-T3** of Cluster T in the Axiom evolution plan
(`giggly-imagining-moore.md`, "Cluster T → INC-T3"): independent of
Cluster A, of INC-T1 (`cmm` replaces `graphify`/`codegraph`, closed), and
of INC-T2 (RTK skill-invoked + `axiom-terminal-output-efficient`, closed).
Explicitly user-approved; low-risk content/policy increment, kept
proportional (no over-build).

## Context

- **Caveman is an external tool, not an Axiom dependency.** Confirmed via
  the already-closed `INC-20260708-caveman-terse-comms-skill` (AB9): "an
  external tool that rewrites agent OUTPUT into a terse 'caveman' notation
  ... Axiom does not vendor or install this tool." `axiom.config/
  toolchain-catalog.yaml` already declares `caveman` (`id: caveman`, `kind:
  output-optimizer`, `mvp: false`) as a pre-existing, optional P1 tool with
  zero runtime wiring anywhere in Axiom — untouched by this increment.
  `packages/cavekit-discipline` is a **different, native** drift-checker
  (spec/context/code keyword-consistency, Spec 0015) — confirmed by
  reading its source (`check.ts`'s `checkDrift`); it has no relation to
  Caveman's concision philosophy and was not touched or conflated here.
- **Pre-existing, narrower prior art found during discovery:
  `axiom-terse-comms`.** `INC-20260708-caveman-terse-comms-skill` (AB9,
  closed) already added a skill that borrows Caveman's philosophy, but
  scoped **strictly** to Axiom-agent-to-Axiom-agent traffic (handoffs,
  subagent briefs, structured status) via a curated symbol legend. It is
  registered through a **completely different mechanism**: a bundled seed
  catalog in `apps/cli/src/commands/workspace-skills.ts`
  (`CANONICAL_SEED_SKILLS`/`CANONICAL_SEED_SOURCES`, hash via
  `computeSkillBundleHash`, materialized only when scaffolding brand-new
  repos), **not** the main `axiom.config/skills-catalog.yaml` (the
  19-entry, doctor-`TC-010`-governed catalog this increment's brief
  requires). Critically, `axiom-terse-comms` **explicitly and permanently
  excludes** anything human-facing (specs, READMEs, PR/commit text,
  user-facing summaries) — it is opt-in (`available`, gated by
  `interAgentTerseComms`), inter-agent-only. This is the opposite scope
  axis from what this increment's brief asks for (a *general* writing
  discipline that explicitly governs human-facing responses too). See
  Assumptions for why this increment adds a **new**, separate,
  catalog-registered skill instead of widening `axiom-terse-comms`.
- **Existing discipline-skill pattern** (`axiom-structured-doubts`,
  `axiom-plan-drift-alignment`, `axiom-functional-checklist-coverage`,
  `axiom-role-close-doc`, and the just-closed `axiom-terminal-output-
  efficient`): plain-heading structure (`# Title` / `## Qué hace` / `##
  Cuándo se usa` / `## Cómo lo hace` with `###` sub-sections / `## Reglas
  absolutas` (Prohibido/Obligatorio) / `## Relación con otras piezas` /
  `## Estado`), registered in `axiom.config/skills-catalog.yaml` with a
  `bundleHash` = sha256 of the source's raw bytes (verified by doctor's
  `TC-010`), and referenced by id from other skills rather than
  reimplemented. None of these discipline skills' "Reglas absolutas" are
  enforced by doctor/code — the skill text itself is the enforceable
  contract, same precedent this increment follows.
- `axiom-terminal-output-efficient.md` (INC-T2) already carried a
  deliberate one-sentence forward reference: "independiente de la futura
  política de concisión transversal para la redacción de las propias
  skills (INC-T3)" — anticipating this exact increment. Closing that loop
  (naming the concrete new skill id) is in scope as a light-touch
  cross-link, per the brief's point 3.
- `Axiom.Spec/specs/manuales/13_Skills_Agentes_y_Roles.md` is the "skills
  manual" the brief referred to ("doc en manuales"); it already documents
  the "Disciplinas transversales" table this new skill belongs in.

## Scope

- **New skill `axiom-concision-discipline`** (id follows the existing
  `axiom-*` convention; name in the catalog: "Concision Discipline",
  matching how sibling discipline entries drop the `axiom-` prefix from
  `name`, e.g. `axiom-structured-doubts` → "Structured Doubts"). Authored
  at `axiom.spec/target-axiom-skills/axiom-concision-discipline.md`,
  mirroring the dominant discipline-skill structure.
- **Skill body** encodes exactly the principles required by the brief:
  not repeating the user's request back; dropping ceremonial intros/
  outros; not narrating trivial operations; presenting the conclusion
  first; a hard "Regla de oro" that concision must NEVER hide information
  needed to validate the work; and an explicit, unbounded-detail carve-out
  for risk, uncertainty, or need for human review, plus evidence,
  constraints, errors, and relevant reasoning that must always survive.
  Explicitly states Caveman is adopted as a PHILOSOPHY only — not
  installed, not vendored, no runtime dependency — and cross-references
  the pre-existing `caveman` toolchain-catalog entry as unrelated/
  untouched.
- **Catalog registration**: `axiom.config/skills-catalog.yaml` gained one
  entry (`axiom-concision-discipline`, `version: 0.1.0`, `status:
  approved`, `securityCheckStatus: ok`, `bundleHash` = sha256 of the raw
  source bytes, computed with Node and verified against every entry's
  live source in the repo, matching what doctor's `TC-010` independently
  recomputes).
- **Guard-test lockstep** (the bundleHash gotcha, same pattern INC-T1/T2
  used): `packages/skills/tests/catalog.test.ts` (the real-catalog
  integration test: 19 → 20 entries, sorted id list updated, header
  comment updated to "6 escenarios") and `packages/doctor/tests/
  skills.test.ts` (the real-repo `TC-010` smoke test: description and
  `/19\/19/` → `/20\/20/` regex updated). A new, dedicated `Scenario 6`
  test block was added to `catalog.test.ts` asserting the new entry's
  metadata (name, version, source, status, securityCheckStatus) and a
  bundleHash RE-COMPUTED from the live source file in the test itself
  (not hardcoded), plus a second test asserting the source body literally
  contains the required principles (no repetir la petición, ceremoniales,
  operaciones triviales, conclusión primero, evidencia, riesgo,
  incertidumbre, revisión humana, the never-hide-validation-info rule,
  "Caveman" + "no se instala", and cross-references to both sibling
  concision-adjacent skills).
- **Light-touch cross-link, closing INC-T2's forward reference**:
  `axiom-terminal-output-efficient.md`'s existing "independiente de la
  futura política de concisión transversal ... (INC-T3)" sentence was
  updated to name the concrete new skill id
  (`axiom-concision-discipline`) instead of forward-referencing an
  unnamed future increment. Because this changed that file's bytes, its
  `bundleHash` in the catalog was recomputed and updated in the SAME
  lockstep pass (verified against every catalog entry, not just the two
  touched ones).
- **Manual doc entry**: `Axiom.Spec/specs/manuales/
  13_Skills_Agentes_y_Roles.md`'s "Disciplinas transversales" table gained
  one row for `axiom-concision-discipline`, in the same style as the
  existing rows, explicitly naming its distinction from both sibling
  skills.
- **No changes to `axiom-terse-comms`/`workspace-skills.ts`**: the brief's
  file scope is `axiom.spec/target-axiom-skills/*` + the skills manual;
  `workspace-skills.ts` is a different mechanism with its own
  drift-guarded test (`apps/cli/tests/terse-comms-drift.test.ts`) that
  hard-asserts exact embedded-constant equality. The new skill only
  documents the distinction in prose (read-only awareness), requiring no
  edit to that file or its tests.

## Non-goals

- Installing, vendoring, or wiring any runtime for Caveman. The
  pre-existing `caveman` entry in `axiom.config/toolchain-catalog.yaml`
  (`kind: output-optimizer`, `mvp: false`) is untouched.
- Modifying `axiom-terse-comms` or its seed-catalog mechanism
  (`apps/cli/src/commands/workspace-skills.ts`,
  `packages/document-bootstrap/src/inter-agent-comms.ts`) — untouched,
  including its drift test.
- `cmm`/providers (INC-T1, closed), RTK runtime invocation
  (INC-T2, closed) — untouched beyond the single cross-link sentence
  described above.
- AutoSkills lock hygiene (INC-S1), worktrees (Cluster W), the unified
  `<project>.axiom` MCP (Cluster A, done) — untouched.
- A `workspace-process-surfaces.ts` TS-constant for the new skill:
  confirmed (same finding INC-T2 already documented) that only the 8
  role-materialized "process-flow" skills get a condensed TS-constant
  body there; discipline/capability skills like this one rely on
  `@axiom/skills`'s catalog-only materialization path. No entry was added.
- An `agents-catalog.yaml` entry: confirmed skills and agents are not 1:1
  in this product; only the same 8 process-flow skills have a matching
  agent wrapper. No agent was added for this discipline skill, consistent
  with every other discipline skill in the catalog.
- Migrating Axiom's own repos (`Axiom.SDD`/`Axiom.Spec`) — out of scope
  (deferred to the user per the plan).
- Rewriting any existing skill body wholesale — only one pre-existing
  sentence (in `axiom-terminal-output-efficient.md`) was touched, to
  close its own deliberate forward reference.

## Acceptance criteria

- [x] A new shared discipline skill exists (`axiom-concision-discipline`),
      authored at `axiom.spec/target-axiom-skills/
      axiom-concision-discipline.md`, matching the `axiom-*` id
      convention and the dominant discipline-skill structure.
- [x] The skill is registered in `axiom.config/skills-catalog.yaml` with a
      `bundleHash` that is the real sha256 of the source's raw bytes,
      verified to match what doctor's `TC-010` independently recomputes
      (checked for ALL 20 entries, not just the new one).
- [x] The skill body teaches: not repeating the user's request back; no
      ceremonial intros/outros; not narrating trivial operations;
      presenting the conclusion first; PRESERVING detail under risk,
      uncertainty, or need for human review; preserving evidence,
      constraints, errors, and relevant reasoning; and the absolute rule
      that concision must never hide information needed to validate the
      work.
- [x] The skill/spec explicitly states Caveman's philosophy is adopted
      while the tool is NOT installed/vendored/a runtime dependency —
      mirroring INC-T2's RTK-has-no-runtime framing and
      `cavekit-discipline`'s native (not Caveman-derived) status.
- [x] Light-touch cross-links added: from the new skill to
      `axiom-terminal-output-efficient` (terminal-output axis, distinct)
      and to `axiom-terse-comms` (inter-agent-shorthand axis, distinct,
      different catalog mechanism); and INC-T2's own forward-reference
      sentence closed to name the concrete new skill id.
- [x] `packages/skills/tests/catalog.test.ts` and `packages/doctor/tests/
      skills.test.ts` (the two guard tests hard-asserting the real
      catalog's skill count/ids) are updated to 20 in lockstep with the
      new entry.
- [x] A new, dedicated unit test asserts the new skill is in the catalog
      with a valid bundleHash, the expected metadata, and the core
      principles literally present in the source text.
- [x] `npm run build` (`tsc -b`) passes.
- [x] Targeted `vitest run` for `packages/skills`, `packages/doctor`, plus
      the related `apps/cli` skills/toolchain/terse-comms-drift files and
      `packages/toolchain`, passes with 0 new failures.
- [x] Full-repo `vitest run` passes with the exact expected delta (INC-T2's
      documented baseline of 297 files / 2992 tests, plus exactly the 2
      new tests this increment added: 297 files / 2994 tests, 0
      failures) — confirms zero regressions anywhere else in the repo.
- [x] A manual/doc entry was added to the repo's skills manual
      (`Axiom.Spec/specs/manuales/13_Skills_Agentes_y_Roles.md`).
- [x] No `git add`/`commit`/`push` was run; the working tree is left for
      review.

## Open questions

None blocking.

## Assumptions

Each is a deliberate, narrowest-change resolution of an ambiguity the
brief left open, using existing-precedent-first reasoning (no stop-and-ask
per the brief's guardrails):

1. **Skill id**: `axiom-concision-discipline` (not the brief's other
   offered alternative, `axiom-concise-communication`) — "discipline"
   matches the manual's own section header for this class of skill
   ("Disciplinas transversales") and the naming pattern of its closest
   catalog siblings (`axiom-structured-doubts`, `axiom-plan-drift-
   alignment`, `axiom-functional-checklist-coverage`), all noun phrases
   naming the discipline itself rather than a verb-based framing.
2. **New, separate catalog entry — not a widened `axiom-terse-comms`**:
   discovered during discovery (not named in the brief's "Read first"
   list) that `axiom-terse-comms` already borrows Caveman's philosophy,
   but (a) through a structurally different mechanism (bundled seed in
   `workspace-skills.ts`, own hash function, own drift test) than the one
   the brief explicitly requires (`axiom.config/skills-catalog.yaml` +
   doctor `TC-010`), and (b) with the opposite scope on the
   human-facing axis (terse-comms explicitly EXCLUDES anything a human
   reads; this brief's principles — conclusion-first, no ceremonial
   intros/outros, no repeating the request — are explicitly ABOUT
   human-facing communication). Treating them as the same skill would
   either silently narrow the new policy to agent-only traffic (wrong,
   contradicts the brief) or silently widen `axiom-terse-comms`'s
   explicit, tested "never compress human-facing" boundary (a breaking,
   out-of-scope change to a closed increment). A new, complementary skill
   with an explicit cross-link in both directions was the narrowest
   change consistent with both artifacts' existing contracts.
3. **Skill body structure**: plain headings, matching every other
   discipline skill (100% of the reusable/discipline class uses this
   form; the one YAML-frontmatter outlier, `axiom-context-persistence`,
   is a platform meta-skill, not a discipline skill, and no discipline
   skill has ever followed it).
4. **No `workspace-process-surfaces.ts` TS-constant, no `agents-catalog.yaml`
   entry**: re-confirmed the same findings INC-T2 already documented for
   its own skill (only 8 process-flow skills get a TS-constant/agent
   pairing) — this discipline skill follows the identical catalog-only
   pattern as every other non-process-flow skill.
5. **`axiom-terminal-output-efficient.md` cross-link + bundleHash
   relockstep**: chosen over leaving the stale "futura política...
   (INC-T3)" forward reference, because that sentence explicitly named
   this very increment and closing it costs one sentence + one hash
   recompute (a known, low-risk, already-proven lockstep pattern) versus
   leaving a dangling forward-reference in a closed increment's shipped
   skill. The change is prose-only (no semantic/behavioral change to that
   skill's policy), so it carries no test risk beyond the hash itself,
   which was verified against every catalog entry, not assumed.
6. **`axiom-terse-comms`/`workspace-skills.ts` left untouched**: the
   brief's explicit file scope is `axiom.spec/target-axiom-skills/*` +
   the skills manual; that seed mechanism lives outside both, has its own
   hard drift test, and the brief's guardrails direct minimal, focused
   changes. The distinction is documented in prose from the new skill
   instead (a read-only cross-reference, zero risk to the existing
   drift-guarded content).
7. **`name` field drops the `axiom-` prefix** in the catalog entry
   ("Concision Discipline", not "Axiom Concision Discipline") — matches
   every other discipline-skill catalog entry's `name` field exactly.

## Implementation notes

All files changed live in `Axiom/` (the product monorepo) unless noted.

- New: `axiom.spec/target-axiom-skills/axiom-concision-discipline.md`.
- Changed: `axiom.config/skills-catalog.yaml` (new entry + explanatory
  comment; also the `axiom-terminal-output-efficient` entry's
  `bundleHash`, recomputed after its one-sentence cross-link edit),
  `axiom.spec/target-axiom-skills/axiom-terminal-output-efficient.md`
  (one-sentence cross-link closing its own INC-T3 forward reference),
  `packages/skills/tests/catalog.test.ts` (19→20 + new `Scenario 6` block,
  2 new tests, header comment updated), `packages/doctor/tests/
  skills.test.ts` (19→20 description + regex).
- Changed (`Axiom.Spec/`, this sibling spec repo): `specs/manuales/
  13_Skills_Agentes_y_Roles.md` (new row in "Disciplinas transversales").
- Explicitly NOT changed (see Scope/Non-goals/Assumptions for why):
  `apps/cli/src/commands/workspace-skills.ts`, `apps/cli/tests/
  workspace-skills.test.ts`, `packages/document-bootstrap/src/
  inter-agent-comms.ts`, `axiom.config/agents-catalog.yaml`,
  `apps/cli/src/commands/workspace-process-surfaces.ts`,
  `axiom.config/toolchain-catalog.yaml` (the `caveman` entry there
  already existed, unchanged), any `packages/cavekit-discipline` file
  (confirmed unrelated — native drift-checker, not a Caveman import).

## Validation

Ran from `C:\repos\Axiom Workspace\Axiom`:

- `npm run build` (`tsc -b`) — PASSED, clean, exit 0, no output.
- Targeted `npx vitest run packages/skills packages/doctor` — **24 files,
  257 tests passed** (up from a 255-test green baseline captured before
  any change; the 2 new `Scenario 6` tests account for the delta). Zero
  failures, zero pre-existing failures to classify (the scope was fully
  green before and after).
- Targeted `npx vitest run apps/cli/tests/workspace-process-surfaces.test.ts
  apps/cli/tests/skills.test.ts apps/cli/tests/workspace-skills.test.ts
  apps/cli/tests/toolchain-catalog-real.test.ts
  apps/cli/tests/terse-comms-drift.test.ts packages/toolchain` — **8
  files, 70 tests passed**, confirming the untouched `axiom-terse-comms`
  drift guard and toolchain catalog are unaffected.
- Full-suite `npx vitest run` (whole repo) — **297 files, 2994 tests
  passed. Zero failures.** Matches INC-T2's documented green baseline (297
  files / 2992 tests) plus exactly the 2 new tests added here — confirms
  no regressions anywhere else in the repo.
- Independent bundleHash audit (ad hoc Node script, not a committed test):
  recomputed sha256 for all 20 catalog entries' live source files and
  diffed against the stored `bundleHash` values — 20/20 match, including
  both entries touched in this increment.

No pre-existing failures were observed to classify — the suite was fully
green before and after in every scope run.

## Result

Implemented. `axiom-concision-discipline` is now a catalog-registered,
Axiom-native discipline skill teaching every skill/agent how to WRITE:
lead with the conclusion, cut ceremony and trivial narration, never
re-state the request — while making preservation of evidence, errors,
constraints, and risk/uncertainty detail an absolute, non-negotiable rule
("concision must never hide information needed to validate the work").
Caveman's philosophy is adopted in prose only: the tool is not installed,
vendored, or given any runtime, exactly like the pre-existing `caveman`
toolchain-catalog entry it explicitly (and correctly) leaves untouched.
Discovery surfaced a narrower, already-shipped prior art
(`axiom-terse-comms`, inter-agent-only, opt-in, different catalog
mechanism) that this increment deliberately did not touch, widen, or
duplicate — instead cross-referencing it explicitly so the two are never
conflated. The catalog's `bundleHash` gotcha was honored for both the new
entry and the one pre-existing entry whose prose was intentionally
touched (`axiom-terminal-output-efficient`, closing its own forward
reference); both hardcoded-count guard tests were updated in lockstep,
plus a new dedicated test guarding both the new entry's metadata and its
policy content. A manual doc entry was added. Build + full test suite are
green (297/297 files, 2994/2994 tests). No git operations were performed.

## General spec integration

Integrado en la spec canónica al cierre del batch INC-20260724-* (integración
única de toda la tanda). Ficheros que recibieron el conocimiento de este
incremento:

- `06_Integraciones_y_Capacidades.md` — skill `axiom-concision-discipline`
  (filosofía de concisión de Caveman sin instalar Caveman; catálogo 19→20;
  cross-links a `axiom-terse-comms`/`axiom-terminal-output-efficient`).
- `08_Glosario.md` — `axiom-concision-discipline` (skill).
- `01_Requisitos_Funcionales.md` — RF-AXM-045.

(La fila en `manuales/13_Skills_Agentes_y_Roles.md` la escribió el propio
incremento; no forma parte de la spec general `00–08`.)
