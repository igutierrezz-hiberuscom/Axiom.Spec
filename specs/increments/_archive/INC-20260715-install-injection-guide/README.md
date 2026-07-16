# Increment: install-injection-guide

Status: closed
Date: 2026-07-15

## Goal

Document, in the practical manuals (`Axiom.Spec/specs/manuales/`), (a) the full
SDD surface/skill/agent set Axiom installs per repo role, and (b) the
per-project injection channel teams use to add their own stack-specific depth
— so a generic Axiom install can carry the same richness a role-specialized
system like KVP25 has, without baking stack rules into the product itself.
Closes M8 of the KVP25-alignment review.

## Context

Axiom materializes process surfaces per repo role via
`surfaceIdsForRole(repoRole)` in
`Axiom/apps/cli/src/commands/workspace-process-surfaces.ts`:
`spec` -> `axiom-spec-author`, `axiom-role-planner`, `axiom-spec-integrator`,
`axiom-tech-context`; `code` -> `axiom-role-implementer`; `sdd` ->
`axiom-sdd-orchestrator`, `axiom-phase-reviewer`, `axiom-qa-validator`. On top
of that, `axiom.config/skills-catalog.yaml` (18 skills) and
`axiom.config/agents-catalog.yaml` (14 agents) hold the full catalog,
including 4 reusable cross-cutting discipline skills added in this batch
(`axiom-structured-doubts`, `axiom-functional-checklist-coverage`,
`axiom-plan-drift-alignment`, `axiom-role-close-doc`) that the flow surfaces
reference by id rather than duplicate.

Nothing in `manuales/` (01–12) currently explains this installed
surface/skill/agent map, nor how a project injects its own stack-specific
depth (architecture patterns, UI conventions, permission/multi-tenant rules,
build/test commands) without that depth being hardcoded into the generic
Axiom product. This gap was flagged as M8 in the KVP25-alignment review: a
role-specialized system like KVP25 gets richness by hardcoding stack rules
into per-role agents; Axiom instead keeps the product generic/adapter-agnostic
and expects that depth to live in project data
(`axiom.config/skills-index/<role>.yaml`, the project's technical context via
`axiom-tech-context`, and project role skills) — but that contract was never
written down for operators.

## Scope

- New manual page `Axiom.Spec/specs/manuales/13_Skills_Agentes_y_Roles.md`
  covering: what Axiom installs per repo role (table from
  `surfaceIdsForRole`); the 4 cross-cutting discipline skills +
  `axiom-phase-reviewer`; a brief tour of the catalog agents; the per-project
  injection channel (contrasted explicitly against a role-specialized
  hardcoded-stack-rules system); and an end-to-end cycle through the
  review/QA/security gates (marked optional where applicable).
- Link the new page from `Axiom.Spec/specs/manuales/README.md`'s index.
- Docs-only; no product code changes in `Axiom/`.

## Non-goals

- No changes to `Axiom/` product code, catalogs, or `workspace-process-surfaces.ts`.
- No edits to `Axiom.Spec/specs/00_*.md`…`08_*.md` (deferred to the
  orchestrator's cross-increment pass).
- No invention of ids, file names, or config paths not already verified
  against the catalogs / `surfaceIdsForRole` / `role-index.ts` in `Axiom/`.
- No enterprise lifecycle constructs.

## Acceptance criteria

- [x] `13_Skills_Agentes_y_Roles.md` exists under `Axiom.Spec/specs/manuales/`
      and is linked from `manuales/README.md`'s index.
- [x] Content accurately reflects the installed surfaces/skills/agents,
      grounded in `skills-catalog.yaml`, `agents-catalog.yaml`, and
      `surfaceIdsForRole` in `workspace-process-surfaces.ts` (no invented
      ids/paths).
- [x] The injection-channel section explains
      `axiom.config/skills-index/<role>.yaml`, project technical context
      (`axiom-tech-context`), and project role skills as the concrete
      per-project extension points, contrasted against a role-specialized
      hardcoded-stack-rules system.
- [x] All relative links in the new page resolve to existing files.
- [x] No product code changed in `Axiom/` as part of this increment.

## Open questions

None — self-contained, docs-only brief with all ids/paths pre-verified
against the catalogs and `workspace-process-surfaces.ts` before writing.

## Assumptions

- The 4 discipline skills (`axiom-structured-doubts`,
  `axiom-functional-checklist-coverage`, `axiom-plan-drift-alignment`,
  `axiom-role-close-doc`) are catalog skills referenced BY the process
  surfaces (by id, in their bodies) rather than surfaces materialized
  directly per repo role themselves — confirmed by reading their sources
  under `Axiom/axiom.spec/target-axiom-skills/` and the surface bodies in
  `workspace-process-surfaces.ts`.
- `axiom-role-close-doc` is invoked implicitly right after a role's technical
  close (per its own source and the `axiom-spec-integrator` source, which
  lists it as a precondition/consumed source), not as its own installed
  surface — documented as part of the end-to-end cycle, not the surface
  table.
- `axiom.config/skills-index/<role>.yaml` does not exist as a static example
  file in the `Axiom` product repo itself (it is generated per-project at
  materialization time by `augmentSkillsIndex` in
  `workspace-process-surfaces.ts`, reusing the schema in
  `packages/skills/src/role-index.ts`); the manual describes the mechanism
  and its schema location, not a concrete file that should exist in `Axiom/`.

## Implementation notes

Read `Axiom/axiom.config/skills-catalog.yaml`,
`Axiom/axiom.config/agents-catalog.yaml`,
`Axiom/apps/cli/src/commands/workspace-process-surfaces.ts` (full file,
including all `SURFACES` bodies and `surfaceIdsForRole`),
`Axiom/packages/skills/src/role-index.ts` (skills-index schema/path
convention), the 4 discipline-skill sources plus `axiom-tech-context.md` and
`axiom-security-reviewer.md` under `Axiom/axiom.spec/target-axiom-skills/` and
`target-axiom-agents/`, and the sibling manuals (`README.md`, `02`, `04`,
`05`, `07`, `08`, `09`, `10`) for style/format/link consistency before
writing. Wrote `13_Skills_Agentes_y_Roles.md` in Spanish, matching the
concise/task-oriented tone of the existing manuales, and added its entry to
`manuales/README.md`'s index under a new "Superficies instaladas" subsection
alongside the existing groupings.

## Validation

No validation command was found. Performed best-effort validation by
inspecting changed files and checking consistency against the requested
behavior.

Link-existence check for `13_Skills_Agentes_y_Roles.md`'s `## Relacionado`
section (all relative to `Axiom.Spec/specs/manuales/13_Skills_Agentes_y_Roles.md`):

- `05_Incrementos.md` -> exists
- `07_Planes.md` -> exists
- `08_Implementacion.md` -> exists
- `09_Revisiones.md` -> exists
- `10_Archivado.md` -> exists
- `11_Launcher_Visual.md` -> exists
- `02_Configuracion.md` -> exists
- `04_Generar_Spec_y_Contexto_Tecnico.md` -> exists
- `../05_Interfaces_Operativas.md` -> resolves to
  `Axiom.Spec/specs/05_Interfaces_Operativas.md`, exists

`manuales/README.md`'s new index entry `[13. Skills, agentes y roles instalados](13_Skills_Agentes_y_Roles.md)`
resolves to the new file in the same directory.

Confirmed via `git status --porcelain` in `Axiom/` before and after this
increment's work: identical output (42 pre-existing modified/untracked lines
from prior increments in this batch) — no new files or modifications
introduced in `Axiom/` by this increment.

## Result

Closed. Created
`Axiom.Spec/specs/manuales/13_Skills_Agentes_y_Roles.md`, covering the
installed surface/skill/agent map per repo role (grounded in
`surfaceIdsForRole` + both catalogs), the 4 cross-cutting discipline skills +
`axiom-phase-reviewer`, a short tour of catalog agents, the per-project
injection channel (`skills-index/<role>.yaml`, technical context via
`axiom-tech-context`, project role skills) contrasted against a
role-specialized hardcoded-stack-rules system, and an end-to-end gated cycle
from spec-author through spec-integrator. Linked it from
`manuales/README.md`'s index. All relative links verified to resolve; no
product code in `Axiom/` was touched by this increment.

## General spec integration

Integrado por el pase final de la orquestación (autopilot, 2026-07-15):
- `01_Requisitos_Funcionales.md` → **RF-AXM-039**.
- `03_Modelo_Operativo_y_Datos.md` → sección "Superficies SDD por rol de repo + canal de inyección por proyecto".
- `00_Resumen_Ejecutivo.md` → mención del canal de inyección en la tanda de alineación.
- `08_Glosario.md` → término "Canal de inyección por proyecto".
- Cross-referenciado desde `04`/`05`/`06`/`07`.

Archivado en `Axiom.Spec/specs/increments/_archive/`.
