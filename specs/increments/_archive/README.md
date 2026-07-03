# Archive — 2026-07-02/03 Axiom redesign roadmap

This folder holds the completed increment chains from the Axiom
architecture redesign roadmap (`INC-20260702-*`), which reconciled
Axiom's product architecture against two external decision documents
(`axiom_decisiones_sesion_prompt_implementacion.md` and
`axiom_decisiones_sesion_addendum_revision.md`), plus the D3 side-quest
(`axiom.yaml schemaVersion: 2` re-enablement).

## What is canonical vs. historical

- **`Axiom.Spec/specs/00_Resumen_Ejecutivo.md` through `08_Glosario.md`
  are the canonical, current-state reference.** They were updated, by
  subject (repo topology, artifact model, MCP tool layer, doctor checks,
  versioning, deferred work, etc.), after every increment in this roadmap
  closed. Read those files, starting from `Axiom.Spec/specs/README.md`'s
  navigation, to understand the current Axiom architecture — you do not
  need to read anything in this archive to do so. (An intermediate closure
  pass had consolidated this content into a standalone
  `Axiom.Spec/general-spec.md` file instead; that file has since been
  removed and its content redistributed into the 8 topic files, per
  `Axiom.Spec/specs/README.md`'s explicit rule against a separate legacy
  structure kept "solo por continuidad.")
- **This archive is historical detail and audit trail, not a reading
  requirement.** Each folder here is a closed increment chain
  (migration-engineer audit -> design/implementation -> validator-review,
  or a subset of that sequence depending on the increment) with its own
  goal, scope, findings, and closure record. Consult a specific folder
  only if you need the full reasoning, rejected alternatives, or exact
  code-reading trail behind a fact already summarized in the 8 topic
  files. Individual folders' own "General spec integration" sections may
  still mention `general-spec.md` by name — that reflects the state of
  the workspace at the time each chain closed and is left as historical
  record, not corrected retroactively.
- The parent roadmap index,
  `Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/README.md`,
  was intentionally **not** archived — it remains at the top level as the
  master index and closing summary for the whole roadmap, with pointers to
  both the 8 topic files and this archive.

## Contents

71 folders, each a chain step (audit / design / impl / validator / other
role-specific suffix) for one of the roadmap's 23 increments or the D3
side-quest. Every chain closed with a `Status: closed` terminal
validator-reviewer pass, confirmed by direct inspection before archiving.
One exception, also archived here for record-keeping rather than left
loose at the top level: `INC-20260702-tui-menu-promote-inventory-screens`,
a genuinely **not-started, deferred placeholder** increment (not a closed
chain) — see `Axiom.Spec/specs/05_Interfaces_Operativas.md`'s TUI section
for its current status.

Do not add new increments here directly. New work should get a fresh
increment folder at `Axiom.Spec/specs/increments/` (top level) and only
move here once its own chain closes.
