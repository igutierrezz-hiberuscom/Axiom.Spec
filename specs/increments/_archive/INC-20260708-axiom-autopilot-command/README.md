# Increment: Comando /axiom-autopilot para orquestación desatendida multi-incremento

Status: closed
Date: 2026-07-08

## Goal

Codify, as a reusable Claude Code slash command + skill, the exact
multi-increment orchestration playbook that has so far been performed
manually across `Axiom.SDD` / `Axiom.Spec` / `Axiom` sessions (e.g. the
`INC-20260702-*-reconcile` and `INC-20260705-workspace-*` batches): a
future session should be able to type
`/axiom-autopilot <description of all the changes wanted>` and have the
main agent decompose the batch into focused increments, run them
sequentially (or in parallel when file trees are disjoint) via
`axiom-increment` subagents, auto-resolve ambiguity with the most
reasonable option while recording every decision, independently verify
each increment's result, integrate stable knowledge into the canonical
`00–08` spec files at the end, archive every increment, and close with
an explicit decisions summary — with no further prompting.

## Context

The workspace is a 3-repo layout (`Axiom.SDD` implementation/tooling,
`Axiom.Spec` canonical specs, `Axiom` optional/future consumer — this
particular workspace uses `Axiom` as the actual product code and
`Axiom.Spec/specs` as the canonical spec tree with topic files
`00_Resumen_Ejecutivo.md` … `08_Glosario.md`, navigation in
`specs/README.md`, per-increment folders under `specs/increments/<INC-id>/`,
and a `specs/increments/_archive/` for closed/integrated increments).//
`Axiom.SDD/AGENTS.md` defines the canonical lightweight SDD lifecycle
(understand → locate/refine spec → ask only if blocking → plan →
implement → validate → review → close/pending → integrate stable
knowledge). `.claude/commands/axiom-increment.md` +
`.claude/agents/axiom-increment.md` already implement this lifecycle as
a single-increment slash command + subagent pair (there was no
`.claude/skills/` directory yet before this increment — `commands/` and
`agents/` existed, `skills/` did not).

Prior large batches (the `INC-20260702` reconcile wave, the
`INC-20260705` workspace-setup wave) were carried out manually: a human
operator repeatedly invoked the increment workflow per sub-change,
resolved ambiguity turn-by-turn, and performed a final cross-increment
integration pass into `00–08` plus archival. This increment turns that
manual playbook into a self-contained, unattended orchestration
artifact so it can be invoked with a single command in any future
session.

## Scope

- New slash command `.claude/commands/axiom-autopilot.md` (frontmatter
  + body, mirroring the format of `axiom-increment.md`): loads/runs the
  `axiom-autopilot` skill with `$ARGUMENTS` as the batch of requested
  changes; states the primary path (run the playbook in the MAIN
  conversation, spawning `axiom-increment` subagents per increment —
  this is explicitly not a subagent-only flow) and a fallback (inline
  execution if agent launch is unavailable).
- New skill `.claude/skills/axiom-autopilot.md` containing the full
  playbook: baked-in workspace context (repo roles, canonical spec
  layout, known pre-existing test baseline, gotchas), the 9-step
  ordered procedure (restate as numbered change-list → optional
  lightweight `Explore` grounding → decompose into focused sequential
  increments → spawn self-contained `axiom-increment` subagent briefs
  → auto-answer ambiguity and record decisions → independently verify
  each result → integrate stable knowledge into `00–08` at the end →
  archive increments → final decisions summary), and the hard
  guardrails (no mutating git, no speculative architecture, no
  unrelated file changes, never stop to ask).
- Minimal cross-reference additions (one or two lines each) to
  `.claude/commands/axiom-increment.md` and
  `.claude/agents/axiom-increment.md` noting that `axiom-increment` is
  the per-increment executor used by `/axiom-autopilot` for
  batch/unattended runs.
- This spec file.

## Non-goals

- No changes to Axiom product code (`Axiom/**`) — this increment
  delivers Claude Code configuration + documentation only.
- No changes to `axiom-bug.md` or `axiom-review.md` command/agent
  files (out of scope; `/axiom-autopilot` only orchestrates the
  increment lifecycle, not bug fixing or review).
- No `.claude/skills/axiom-increment.md` file is created as part of
  "updating existing files minimally" — no such skill file existed
  before this increment (only `commands/` and `agents/` existed), so
  there was nothing pre-existing to cross-reference there. Creating a
  net-new `axiom-increment` skill file is out of scope for this
  increment (would be scope creep beyond the requested autopilot
  command).
- No skills/commands registry or index file is introduced — none
  existed in `.claude/` before this increment, and creating one is an
  enterprise-lifecycle-style construct not requested here (see
  `Axiom.SDD/AGENTS.md` Explicit Bootstrap Limits: "mandatory indexes").
- No integration into `Axiom.Spec/specs/00_Resumen_Ejecutivo.md` …
  `08_Glosario.md` in this pass — the whole point of `/axiom-autopilot`
  is that the final cross-increment `00–08` integration is something
  the orchestrator itself performs at the end of a batch; this
  increment only builds the orchestrator.
- No actual execution of a multi-increment batch is performed as part
  of closing this increment — this increment's deliverable is the
  reusable command/skill artifact itself, validated by inspection.

## Acceptance criteria

- [x] `.claude/commands/axiom-autopilot.md` exists, has valid
      frontmatter (`--- description: ... ---`), and mirrors the
      structural format of `axiom-increment.md` (primary path /
      fallback path / guardrails).
- [x] `.claude/skills/axiom-autopilot.md` exists and contains: the
      baked-in context (repo roles, canonical spec file layout,
      pre-existing test baseline note, the four named gotchas), the
      full ordered playbook (all 9 steps), and the guardrails section,
      matching the increment prompt's required content essentially
      verbatim in structure.
- [x] The command explicitly states the orchestration runs in the MAIN
      conversation and spawns `axiom-increment` subagents per
      increment (not a subagent-only flow).
- [x] `.claude/commands/axiom-increment.md` and
      `.claude/agents/axiom-increment.md` each carry a short (1-2
      line) cross-reference note about their use by `/axiom-autopilot`
      for batch/unattended runs, with no other content rewritten.
- [x] No files under `Axiom/**` were modified.
- [x] This spec file documents goal/scope/acceptance/decisions and
      states explicitly that `00–08` integration is deferred to the
      orchestrator's own future runs, not performed here.
- [x] Best-effort validation performed and documented (markdown/
      frontmatter consistency, mirroring check against
      `axiom-increment.md`).

## Open questions

None blocking. The increment prompt supplied closed decisions for
naming, execution model (main conversation, not subagent-only), file
locations, and required content. Where the prompt was ambiguous or
silent, resolutions are recorded under Assumptions/Decisions below.

## Assumptions

- **Command name**: kept `axiom-autopilot` as instructed (unattended
  batch increment orchestration) — it reads clearly next to the
  existing `axiom-increment`/`axiom-bug`/`axiom-review` triad and
  needs no renaming.
- **No skills directory existed yet**: `.claude/skills/` did not exist
  before this increment (only `commands/` and `agents/` did, each with
  three files, none named after skills). Created the directory and
  placed only `axiom-autopilot.md` in it, since that is the one skill
  file the increment explicitly requires; did not backfill
  `axiom-increment.md`/`axiom-bug.md`/`axiom-review.md` skill files, as
  that would be unrelated scope creep the prompt did not request (the
  prompt's "mirror the FORMAT of the existing `.claude/commands/
  axiom-increment.md`" instruction was applied to the command file,
  which does exist).
- **No registry/index file exists** in `.claude/` (checked directly:
  only `commands/`, `agents/`, `settings.local.json`). Per the
  increment prompt's own conditional ("if there is a skills/commands
  registry/index file... add the new command there") and per
  `Axiom.SDD/AGENTS.md`'s Explicit Bootstrap Limits (no mandatory
  indexes), no such file was created.
- **Cross-reference notes** were added only to `axiom-increment.md`'s
  command and agent files (the two that exist and are directly reused
  by the autopilot playbook), not to `axiom-bug.md`/`axiom-review.md`,
  since the playbook does not invoke those roles.
- **Validation scope**: since this increment only adds/edits Markdown
  config files consumed by Claude Code (no code, no build, no test
  suite applies), validation is limited to structural/consistency
  inspection, per the increment prompt's own stated validation
  approach.
- **Execution model wording**: the skill and command both state plainly
  that `/axiom-autopilot` runs in the *main* conversation thread and
  spawns `axiom-increment` subagents (Task-tool style delegation) per
  increment — it is not implemented as an `agents/axiom-autopilot.md`
  subagent definition, since the prompt is explicit that the
  orchestration itself is not a subagent-only flow.

## Implementation notes

- `.claude/commands/axiom-autopilot.md`: frontmatter description +
  body instructing the agent to load the `axiom-autopilot` skill,
  treat `$ARGUMENTS` as the full batch of requested changes, and follow
  its playbook end-to-end without stopping to ask the user; documents
  the primary path (main-conversation orchestration spawning
  `axiom-increment` subagents) and an inline fallback; repeats the
  hard guardrails (no mutating git, no unrelated files, no speculative
  architecture, do not stop early).
- `.claude/skills/axiom-autopilot.md`: full playbook skill file,
  structured as: Canonical Rule Source → Objective → Baked-in Context
  (repo roles + canonical file map + pre-existing test baseline +
  gotchas) → The Playbook (9 numbered steps, each expanded with
  concrete sub-instructions mirroring the increment prompt) →
  Guardrails → Output Contract (final decisions summary shape).
- `.claude/commands/axiom-increment.md`: appended a one-line note under
  the description/body noting reuse by `/axiom-autopilot`.
- `.claude/agents/axiom-increment.md`: appended a one-line note near
  the top (Objective section) noting reuse by `/axiom-autopilot` for
  batch/unattended runs.

## Validation

No validation command applies; this increment only adds/edits
Markdown command/skill configuration files (no code, no build, no
test suite touches `.claude/**`). Performed best-effort validation by:

- Inspecting `.claude/commands/axiom-autopilot.md` and
  `.claude/skills/axiom-autopilot.md` for valid Markdown structure and,
  for the command file, valid `--- description: ... ---` frontmatter
  matching the exact style of `axiom-increment.md`/`axiom-bug.md`/
  `axiom-review.md`.
- Confirming the command file's primary/fallback/guardrails shape
  mirrors the existing three command files line-for-line in structure
  (not content).
- Confirming the skill file contains every required section from the
  increment prompt (context, all 9 playbook steps, all guardrails).
- Confirming the two cross-reference edits to `axiom-increment.md`
  (command + agent) are additive one-liners that do not alter any
  existing line.
- Confirming no file under `Axiom/**` was touched (`git status`
  equivalent: only `.claude/**` and `Axiom.Spec/specs/increments/
  INC-20260708-axiom-autopilot-command/**` were created/modified).

## Result

Created `.claude/commands/axiom-autopilot.md` and
`.claude/skills/axiom-autopilot.md` (the latter is a new `skills/`
directory, previously absent from `.claude/`), encoding the full
unattended multi-increment orchestration playbook: decompose the
user's batch into focused, dependency-ordered increments; spawn
self-contained `axiom-increment` subagent briefs sequentially (or in
parallel only across disjoint file trees); auto-resolve ambiguity with
recorded rationale; independently re-verify each increment's build/
test claims; perform a single final `00–08` cross-increment integration
pass; archive every increment folder; and close with an explicit
decisions summary. Added minimal one-line cross-reference notes to
`.claude/commands/axiom-increment.md` and
`.claude/agents/axiom-increment.md` documenting their reuse by
`/axiom-autopilot`. No skills/commands registry file was introduced
(none existed) and no Axiom product code was touched.

## General spec integration

None performed, and none needed. This increment's own scope
explicitly excludes `00–08` integration — the entire point of
`/axiom-autopilot` is to perform that integration itself, once, at the
end of a *future* multi-increment batch it orchestrates. There is no
new stable *product* behavior here to consolidate into
`Axiom.Spec/specs/00_Resumen_Ejecutivo.md` … `08_Glosario.md`: this
increment only adds Claude Code tooling/config
(`.claude/commands/**`, `.claude/skills/**`), which is repository
tooling, not canonical Axiom product/behavior knowledge.

### Integración canónica realizada (pase de round 4, 2026-07-08)

Matización tras el pase de integración de round 4 (orquestador): dado que
`/axiom-autopilot` es tooling de DESARROLLO para construir Axiom (vive en
`.claude/`, no en `Axiom/` runtime), su presencia en la spec canónica es
deliberadamente **ligera** — no una capacidad de producto. El pase de
round 4 añadió una única mención breve, correctamente enmarcada como
tooling de dev/workflow:

- **`04_Flujos_SDD_y_Ciclo_de_Vida.md`**: subsección "Tooling de
  orquestación del workspace: `/axiom-autopilot`" bajo el flujo base de
  dogfooding, describiéndolo como la contraparte de tanda desatendida del
  comando `axiom-increment` por incremento único, con la nota explícita de
  que vive en `.claude/` y no es una capacidad de runtime del producto.
- **`08_Glosario.md`**: término "/axiom-autopilot (orquestación
  desatendida)" añadido en la nueva sección de tanda 4, igualmente
  enmarcado como tooling de desarrollo.

No se integró nada en `00`-`03`/`05`-`07`: no hay comportamiento estable
de PRODUCTO que este incremento introduzca.
