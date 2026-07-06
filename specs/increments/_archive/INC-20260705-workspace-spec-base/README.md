# Increment: Workspace spec repo base scaffolding

Status: closed
Date: 2026-07-06

## Goal

When `runWorkspaceSetup` creates a brand-new SPEC repo (`created ===
true`), scaffold the canonical spec + technical-context BASE into it:
the navigable `specs/README.md` + `specs/00_Resumen_Ejecutivo.md`
through `08_Glosario.md` + `context/TECHNICAL_CONTEXT.md` +
`context/README.md`, plus the empty structural directories
(`specs/increments/`, `specs/bugs/`, `specs/archive/`,
`context/architecture/`, `context/integrations/`,
`context/operations/`, `context/references/`). If the spec repo
already existed, skip entirely (never clobber). Only the spec repo
gets this base — control/role repos are untouched by this step.

## Context

Builds on `INC-20260705-workspace-multirepo-setup-engine`
(`runWorkspaceSetup` in `apps/cli/src/commands/workspace-setup.ts`) and
mirrors the pattern established by `INC-20260705-workspace-sdd-skills`
(`workspace-skills.ts`: bundled seed content as TS constants +
best-effort scaffolder + gated wiring on a repo's own `created` flag).

The canonical spec-repo shape already lives in this workspace's own
`Axiom.Spec/`: `specs/README.md`, `specs/00_Resumen_Ejecutivo.md` …
`08_Glosario.md`, `context/TECHNICAL_CONTEXT.md` +
`context/{architecture,integrations,operations,references}/`,
`specs/{increments,bugs,archive}/`. The BASE TEMPLATES for each of
these topics live in `Axiom.Spec/templates/` (verified by direct
read): `specs-readme-template.md`, `executive-summary-template.md`
(→00), `functional-requirements-template.md` (→01),
`non-functional-requirements-template.md` (→02),
`domain-model-template.md` (→03), `business-flows-template.md` (→04),
`user-interfaces-template.md` (→05), `integrations-template.md` (→06),
`security-template.md` (→07), `glossary-template.md` (→08),
`technical-context-template.md` (→context/TECHNICAL_CONTEXT.md),
`context-readme-template.md` (→context/README.md).

Confirmed gap: there is no scaffolder for a spec-repo base today, and
the templates are not bundled into the product — they only exist in
the sibling `Axiom.Spec/templates/` of this workspace. `tsc -b` does
not copy non-TS assets to `dist`, so a runtime `readFileSync` of a
bundled template directory would fail once installed; the templates
must be bundled as TS string constants (same decision already made and
validated by `workspace-skills.ts`'s seed content).

## Scope

- New `apps/cli/src/commands/workspace-spec-base.ts`:
  - Bundled TS constants with the verbatim content of each base
    template listed above (light placeholder substitution applied
    where a template embeds a project-name-shaped placeholder; none of
    the read templates contained one, so substitution is a no-op in
    practice today but the substitution mechanism itself is
    implemented for forward compatibility).
  - `scaffoldSpecRepoBase(args: { specRepoPath: string; projectName:
    string }): Promise<{ filesCreated: string[]; warnings: string[] }>`
    — best-effort, never throws. Writes, guarded per-file (skip +
    warn if a file already exists, never overwrite):
    - `specs/README.md`
    - `specs/00_Resumen_Ejecutivo.md` .. `specs/08_Glosario.md` (9
      files, each seeded from its mapped template; where the mapped
      template's own filename convention differs from the numbered
      topic name used by this workspace — e.g. template
      `03_Modelo_Dominio_o_Datos.md` vs. topic file
      `03_Modelo_Operativo_y_Datos.md` — the file is still written
      under the numbered topic name with the template's real body
      content, since the template is a content skeleton, not a
      filename contract)
    - `context/TECHNICAL_CONTEXT.md`, `context/README.md`
    - `specs/increments/.gitkeep`, `specs/bugs/.gitkeep`,
      `specs/archive/.gitkeep`, `context/architecture/.gitkeep`,
      `context/integrations/.gitkeep`, `context/operations/.gitkeep`,
      `context/references/.gitkeep` (empty dirs are not representable
      in git without a placeholder file)
- Wire into `runWorkspaceSetup` (`workspace-setup.ts`): after the
  existing best-effort steps (including the skills step), look up the
  spec repo's own `created` flag from `repoResults`; if `true`, call
  `scaffoldSpecRepoBase({ specRepoPath: specRepo.path, projectName:
  axiomYamlName })` best-effort and append its
  `filesCreated`/`warnings`. If the spec repo was not freshly created,
  skip silently.

## Non-goals

- No changes to `axiom init`/`runInit` (single-repo path).
- No modification of this workspace's own canonical
  `Axiom.Spec/specs/00-08` or `context/` files — those are this
  workspace's own docs, not a scaffolding target.
- No integration into `Axiom.Spec/specs/00-08` in this pass (deferred
  to the orchestrator's cross-increment integration pass, matching the
  precedent of the sibling `INC-20260705-workspace-*` increments).
- No fabricated product content: the scaffold is a faithful base (the
  template skeletons plus, where a template is a poor structural fit
  for a standalone file, a minimal valid stub with the correct heading
  and a short "(pendiente de completar)" note) — never invented
  narrative content.

## Acceptance criteria

- [x] `scaffoldSpecRepoBase` on a tmp spec repo produces all of
      `specs/00_Resumen_Ejecutivo.md` .. `specs/08_Glosario.md`,
      `specs/README.md`, `context/TECHNICAL_CONTEXT.md`,
      `context/README.md` — all non-empty.
- [x] The 7 empty structural directories exist (via `.gitkeep`).
- [x] A pre-existing file at one of the target paths is not
      overwritten; a warning is recorded instead.
- [x] Content assertions confirm the bundled template content was used
      (a known heading string from each template appears in the
      corresponding output file).
- [x] `runWorkspaceSetup` on a spec whose spec repo does not exist yet
      (fresh tmp dir, `create: true`) ends up with the full spec base
      in the spec repo.
- [x] `runWorkspaceSetup` on a spec whose spec repo path already
      exists (`create: false`) skips the spec-base scaffolding
      entirely.
- [x] The best-effort spec-base step never fails the overall
      `runWorkspaceSetup` call.
- [x] Existing tests keep passing.
- [x] `npm run build` (tsc -b) clean from `Axiom/`.
- [x] Focused vitest run (`apps/cli`, `packages/document-bootstrap`,
      `packages/technical-context`, `packages/user-workspace`) green
      (new + pre-existing).

## Open questions

None blocking — the increment prompt supplied fully closed design
decisions (module shape, template bundling strategy, per-file
no-clobber guard, wiring gate). Narrow implementation choices are
recorded under Assumptions.

## Assumptions

- None of the 12 read base templates contain an obvious
  `{{placeholder}}`- or bracket-style token to substitute with
  `projectName`; the substitution mechanism is implemented (a single
  `applyProjectName` pass per bundled constant) but is a no-op against
  today's template content — forward-compatible if a future template
  edit introduces a real placeholder.
- Template-to-filename mapping follows the increment prompt's explicit
  list verbatim (e.g. `domain-model-template.md` →
  `03_Modelo_Operativo_y_Datos.md`, `business-flows-template.md` →
  `04_Flujos_SDD_y_Ciclo_de_Vida.md`, `user-interfaces-template.md` →
  `05_Interfaces_Operativas.md`, `integrations-template.md` →
  `06_Integraciones_y_Capacidades.md`, `security-template.md` →
  `07_Gobierno_y_Seguridad.md`) even though the template's own internal
  title differs from the numbered topic filename used by this
  workspace — the template body content is a valid generic skeleton
  for that topic regardless of the internal title line.
- `writeGuardedFile` (`@axiom/document-bootstrap`) is reused for each
  per-file write: it already provides the path-guard + atomic
  tmp-then-rename primitive and a `Result` shape that maps cleanly to
  a per-file skip decision, avoiding a second bespoke atomic-write
  helper (unlike `workspace-skills.ts`, which pre-dates this reuse
  opportunity and declares its own `atomicWrite`). The per-file
  no-clobber check itself (`fs.existsSync` before calling
  `writeGuardedFile`) is scaffoldSpecRepoBase's own responsibility,
  since `writeGuardedFile` always overwrites.
- Gate is strictly on the spec repo's own `created` flag (from
  `repoResults`), computed earlier in the same `runWorkspaceSetup`
  call by `writeOneRepo` — same precedent as
  `INC-20260705-workspace-sdd-skills`'s control-repo gate.

## Implementation notes

- New file `apps/cli/src/commands/workspace-spec-base.ts`:
  - `SPEC_BASE_TEMPLATES: Record<string, string>` — bundled verbatim
    body content per output-relative-path, copied from
    `Axiom.Spec/templates/*` at authoring time.
  - `scaffoldSpecRepoBase({ specRepoPath, projectName })` — iterates
    the bundled file map, per file: skip + warn if the target already
    exists, otherwise write via `writeGuardedFile(specRepoPath,
    relativePath, content)`. Writes `.gitkeep` for the 7 empty
    structural dirs the same way. Wrapped so any unexpected throw
    becomes a single warning and a partial result, never propagating.
- `workspace-setup.ts`: new best-effort block after the existing
  skills-scaffolding block, gated on
  `repoResults.find(r => r.topologyId === specRepo.topologyId)?.created
  === true`, calling `scaffoldSpecRepoBase` and appending
  `filesCreated`/`warnings`.
- No enum duplication, no changes to `runInit`/`init.ts`.

## Validation

From `C:\repos\Axiom Workspace\Axiom`:

- `npm run build` (tsc -b):

  ```
  > axiom-product@0.1.0 build
  > tsc -b
  ```

  Clean, no errors.

- `npx vitest run apps/cli packages/document-bootstrap
  packages/technical-context packages/user-workspace`:

  ```
  Test Files  65 passed (65)
       Tests  594 passed (594)
  ```

  All new tests green: `apps/cli/tests/workspace-spec-base.test.ts` (6
  tests covering file/dir scaffolding, bundled-content assertions,
  no-clobber guard, best-effort resilience, idempotency) and 2 new
  scenarios appended to `apps/cli/tests/workspace-setup.test.ts`
  (Scenario g: fresh spec repo gets the spec base; pre-existing spec
  repo path skips it). Every pre-existing test in the focused scope
  still passes, no regressions.

  Additionally ran `npx vitest run packages/skills` in isolation to
  confirm the known pre-existing failure noted in the sibling
  `INC-20260705-workspace-sdd-skills` increment is unrelated to this
  change: `packages/skills/tests/catalog.test.ts` > "carga el catálogo
  real del repo (axiom.config/skills-catalog.yaml)" still fails
  (`expected 'absent' to be 'ok'`) because this workspace's own root
  `axiom.config/skills-catalog.yaml` does not exist — a pre-existing
  gap untouched by this increment, and outside its mandated validation
  scope.

## Result

Implemented `scaffoldSpecRepoBase`
(`apps/cli/src/commands/workspace-spec-base.ts`), bundling the
verbatim content of the 12 base templates
(`Axiom.Spec/templates/specs-readme-template.md`,
`executive-summary-template.md`, `functional-requirements-template.md`,
`non-functional-requirements-template.md`, `domain-model-template.md`,
`business-flows-template.md`, `user-interfaces-template.md`,
`integrations-template.md`, `security-template.md`,
`glossary-template.md`, `technical-context-template.md`,
`context-readme-template.md`) as TS string constants, each augmented
with a short "(pendiente de completar)" stub under its template
headings so no scaffolded file is left empty. Reused
`writeGuardedFile` (`@axiom/document-bootstrap`) for the atomic
path-guarded write of each file, with a per-file
`fs.existsSync`-based no-clobber guard on top (skip + warning, never
overwrite). Wired as a best-effort final step of `runWorkspaceSetup`,
gated strictly on the spec repo's own freshly-created flag
(`repoResults.find(r => r.topologyId === specRepo.topologyId)?.created
=== true`), mirroring the precedent set by
`INC-20260705-workspace-sdd-skills`'s control-repo gate. Control/role
repos are never touched by this step. Build and focused test suite are
both green; the one failing test in the broader `packages/skills`
suite is confirmed pre-existing and unrelated.

## General spec integration

Integrated into the canonical spec in the round-2 cross-increment pass
(covering this increment and the three sibling round-2
`INC-20260705-workspace-*` increments). Files updated:

- **03_Modelo_Operativo_y_Datos.md** — documented the scaffolded spec-repo
  base structure (`specs/README.md` + `specs/00..08` +
  `context/TECHNICAL_CONTEXT.md`/`README.md` + the `.gitkeep` structural
  dirs) and its spec-repo `created`-gating + per-file no-clobber
  semantics in the new "Artefactos adicionales del setup de workspace"
  subsection.
- **06_Integraciones_y_Capacidades.md** — covered as part of the round-2
  workspace-setup additive steps (best-effort, gated) referenced from the
  multi-adapter/skills subsections.
- **01_Requisitos_Funcionales.md** — extended RF-AXM-024 with the
  "freshly-created Spec repo gets the spec + technical-context base"
  sub-point.
- **04_Flujos_SDD_y_Ciclo_de_Vida.md** — added the "Punto de partida de
  un repo de spec recién creado" note tying the scaffolded base into the
  artifact lifecycle (the `specs/{increments,bugs,archive}` structure).
- **08_Glosario.md** — added "Base de spec (scaffold)".
- **07_Gobierno_y_Seguridad.md** — noted the per-file no-clobber + gated +
  best-effort safety of the spec-base step in the ownership section.
