# Increment: Autopilot playbook reconciles technical context on every run

Status: closed
Date: 2026-07-29

## Goal

Bake a permanent "reconcile the technical context" step into the
`/axiom-autopilot` playbook so that EVERY future batch run — not just
this one, done manually — reviews whether its shipped increments require
updates to the technical-context knowledge base at
`Axiom.Spec/context/**`, and performs those updates as part of its final
cross-increment integration pass (alongside, not instead of, the
existing integration into `Axiom.Spec/specs/00..08`).

## Context

The `axiom-autopilot` skill's playbook already had a step 7 ("Final
cross-increment integration into the canonical spec") that integrates
stable knowledge from a batch into `Axiom.Spec/specs/00_Resumen_Ejecutivo.md`
… `08_Glosario.md`, followed by step 8 (archive every increment) and
step 9 (final decisions summary). Nothing in the playbook told the
orchestrator that `Axiom.Spec/context/**` — the separate technical-context
knowledge base describing "how the product is built today" (entry point
`Axiom.Spec/context/TECHNICAL_CONTEXT.md`, subtrees `architecture/`,
`operations/`, `integrations/`, `references/`) — even exists, let alone
that it needs to stay aligned with the real `Axiom/` code as increments
land. This increment is itself TOOLING/DOCS-only: it edits the two
`.claude/` files that define the playbook, so that this reconciliation
happens automatically on every future run instead of being done by hand.

## Scope

- `.claude/skills/axiom-autopilot.md`:
  - "Baked-in Context → Repository roles and canonical file map": new
    bullet documenting `Axiom.Spec/context/` (entry point, subtrees,
    sourcing/assumption discipline, kept-aligned requirement).
  - Objective list item 5: extended to explicitly include the
    technical-context reconciliation, not only the `00..08` specs.
  - New step **7b. Reconcile the technical context
    (`Axiom.Spec/context/**`)**, placed immediately after step 7 (lettered
    to avoid renumbering steps 8-9): instructs the orchestrator to check
    every shipped increment against the current-state signals the
    context tree tracks (renamed paths, new/removed packages, CLI
    commands, adapters, doctor checks, data model/generated-file map,
    new integrations, onboarding/lifecycle changes), update the
    affected doc(s) in place with cited sources, update
    `TECHNICAL_CONTEXT.md`'s "Última validación" date and index, state
    explicitly when nothing applies, and allow delegation to a subagent
    under the same review discipline as step 7.
  - Step 4's subagent-brief requirements: extended the existing
    "do not touch `00..08`" instruction to also forbid subagents from
    touching `context/**` (reserved for the orchestrator's step 7b).
  - Step 8 (archive): extended the archive-note requirement so each
    archived increment's `## General spec integration` note also
    records which `context/**` file(s) were updated for it (or "none").
  - Step 9 (final summary): added a line reporting what was integrated
    into `context/**` (or that nothing applied), alongside the existing
    `00–08` line.
  - Frontmatter `description` and the fallback-path step-count reference
    updated for consistency with the new 7b step.
- `.claude/commands/axiom-autopilot.md`: extended the "Primary path"
  step 6 to state that the orchestrator also reviews and updates
  `Axiom.Spec/context/**` for the shipped increments, as part of the
  same final pass that integrates into `00_Resumen_Ejecutivo.md`
  through `08_Glosario.md` and archives.

## Non-goals

- Editing `Axiom.Spec/specs/00..08` (no batch is running; nothing to
  integrate).
- Editing `Axiom.Spec/context/**` itself (no batch is running; nothing
  to reconcile — this increment only changes the PLAYBOOK that will
  reconcile it on future runs).
- Editing `Axiom/` (no product code changed).
- Archiving or moving any other increment folder.
- Renumbering the skill's existing steps 1-9 (the new step is lettered
  `7b` specifically to avoid churn, per the brief).

## Acceptance criteria

- [x] `.claude/skills/axiom-autopilot.md` documents `Axiom.Spec/context/`
      in the canonical file map.
- [x] `.claude/skills/axiom-autopilot.md`'s Objective list explicitly
      covers technical-context reconciliation.
- [x] `.claude/skills/axiom-autopilot.md` contains a new step 7b with
      concrete instructions: what to check, how to update, source-citation
      requirement, `TECHNICAL_CONTEXT.md` date/index update, explicit
      "no changes needed" path, and subagent-delegation allowance.
- [x] `.claude/skills/axiom-autopilot.md` step 8's archive-note
      requirement now also covers recording the `context/**` outcome.
- [x] `.claude/skills/axiom-autopilot.md` step 9's final summary now
      also reports the `context/**` integration outcome.
- [x] `.claude/commands/axiom-autopilot.md`'s primary path explicitly
      mentions reviewing/updating `Axiom.Spec/context/**` before/as part
      of archiving.
- [x] No existing guardrail was weakened; no unrelated file was touched.
- [x] Increment spec created under
      `Axiom.Spec/specs/increments/INC-20260729-autopilot-technical-context-step/README.md`.

## Open questions

None blocking.

## Assumptions

1. **"7b" lettering is correct per the brief's explicit instruction**
   ("renumber nothing else, or letter it 7b to avoid churn") — steps 8
   and 9 keep their original numbers; 7b sits between 7 and 8 in reading
   order.
2. **The technical-context tree's shape (`TECHNICAL_CONTEXT.md` +
   `architecture/`/`operations/`/`integrations/`/`references/`) is
   stable enough to bake into the playbook as "assume this; do not
   rediscover it every run"** — confirmed by reading
   `Axiom.Spec/context/TECHNICAL_CONTEXT.md` directly before writing the
   new bullet/step, matching its own stated structure, index, and
   "Regla crítica" verbatim in spirit (not copied wholesale).
3. **No code/build validation applies** — this is a Markdown-only,
   tooling/docs increment (no `package.json`, no build/test tooling
   exists for `.claude/` skill/command files); validation is a
   consistency re-read, per the brief.

## Implementation notes

The new step 7b is deliberately modeled on the shape of the existing
step 7 (same "review every shipped increment", "cite sources",
"delegate-with-review" structure) so the orchestrator applies the same
rigor to both the specs and the technical context, rather than treating
the context reconciliation as a lesser afterthought. The "current-state
signals" checklist in 7b was derived directly from
`Axiom.Spec/context/TECHNICAL_CONTEXT.md`'s own stated scope (package/
command/adapter counts, doctor GATEs, data model, integrations,
onboarding/lifecycle) so the check is concrete rather than open-ended.

## Validation

No validation command was found for `.claude/skills/*.md` or
`.claude/commands/*.md` (Markdown playbook/prompt files — no
package.json, build, or test tooling applies to this repository layer).
Performed best-effort validation instead:

- Re-read `.claude/skills/axiom-autopilot.md` in full after all edits.
  Confirmed: the new `context/` bullet appears in the canonical file
  map (between the `specs/` and `Axiom.SDD/` bullets); Objective item 5
  now names both `00..08` and `context/**`; step 7b appears intact
  between steps 7 and 8, with no renumbering of 8/9; step 8's archive
  bullet now requires recording the `context/**` outcome; step 9 has a
  new bullet for the `context/**` integration outcome; step 4's
  "do not integrate" instruction now also forbids touching `context/**`;
  the frontmatter `description` mentions the technical-context
  reconciliation.
- Re-read `.claude/commands/axiom-autopilot.md` in full after edits.
  Confirmed: primary-path step 6 now states the orchestrator reviews/
  updates `Axiom.Spec/context/**` as part of the final pass, before/
  alongside archiving; the fallback path references "steps 1-9,
  including 7b"; all existing guardrails (no mutating git commands, no
  speculative architecture, no unrelated-file edits, no stopping to
  ask) are unchanged and still present verbatim.
- Confirmed (via directory listing) that no files were touched outside
  `.claude/skills/axiom-autopilot.md`, `.claude/commands/axiom-autopilot.md`,
  and this increment's own spec folder — `Axiom/`, `Axiom.Spec/specs/00..08`,
  and `Axiom.Spec/context/**` were not modified, and no increment folder
  was moved or archived.

## Result

Implemented. Both `.claude/` playbook files now bake in a permanent
technical-context reconciliation step: the skill gained a documented
`Axiom.Spec/context/` file-map entry, an extended objective, a new step
7b with concrete review/update/citation/delegation instructions, an
extended archive-note requirement (step 8), and an extended final-summary
requirement (step 9); the command wrapper's primary path now states the
same expectation in its step 6. Every future `/axiom-autopilot` run will,
by default, review and update `Axiom.Spec/context/**` for its shipped
increments as part of its existing final integration pass — mirroring,
and making permanent, what this run's own final pass will do manually
for the batch it is part of.

## General spec integration

Integrated into `Axiom.Spec/specs/04_Flujos_SDD_y_Ciclo_de_Vida.md`: the
autopilot workflow now explicitly performs the `Axiom.Spec/context/**`
reconciliation before archiving and reports the outcome. Technical-context
outcome: none for this tooling-only increment; the context update belongs to
`INC-20260729-technical-context-reconciliation`.
