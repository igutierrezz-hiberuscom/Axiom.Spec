# Increment: spec manuales

Status: closed
Date: 2026-07-15

## Goal

Create an "Axiom manuals" section (`Axiom.Spec/specs/manuales/`) that explains,
for a team that just had Axiom installed across their repos: what each thing
is, how to configure/change config, how to update versions, how to generate
the spec or technical context, and how to run increments, bugs, plans,
implementation, reviews, and archiving — plus the peculiarities of what was
installed for THIS workspace (the launcher's visual front, and the Azure
DevOps tracker plugin: what to fill when creating increments/bugs and where
the API key/PAT goes). Docs-only, no product code changes.

## Context

`Axiom.Spec/specs/` already holds the canonical 00–08 spec files plus
`increments/`, `bugs/`, `archive/`. There was no user-facing, spec-side
how-to layer aimed at a team member who just cloned/installed Axiom and needs
practical guidance (as opposed to the exhaustive functional/technical detail
in 00–08). The product repo (`Axiom/docs/`) already has an operator-facing
doc tree (`installation.md`, `configuration/`, `cli/*.md`) that this manual
set reuses facts from without copying verbatim. Four increments closed
earlier the same day
(`INC-20260715-adapter-agent-tuning`, `INC-20260715-launcher-ado-bridge`,
`INC-20260715-launcher-doctor-gate`, `INC-20260715-launcher-onboarding`)
shipped the launcher visual front's doctor gate, onboarding tab, ADO
work-item-creation bridge, and per-adapter agent tuning — all of which this
manual set documents from the spec side.

## Scope

- New folder `Axiom.Spec/specs/manuales/` with 13 Markdown pages: an index
  (`README.md`) plus 12 topic pages (`01`–`12`), each with a title, one-line
  purpose, substantive how-to content, and a "## Relacionado" footer of
  relative cross-links.
- One navigation entry added to `Axiom.Spec/specs/README.md` pointing to
  `manuales/README.md`.
- This increment's own spec file.

## Non-goals

- No changes to `Axiom/` (product code).
- No edits to `Axiom.Spec/specs/00_*.md` … `08_*.md` (manuals may link to
  them; cross-referencing 00–08 → manuales is deferred to the orchestrator's
  final pass).
- No invented CLI commands, config fields, or URLs — every command/field
  named in the manuals was verified against `Axiom.Spec/specs/05_*.md`,
  `08_*.md`, `04_*.md`, the four `INC-20260715-launcher-*`/`adapter-agent-tuning`
  increment specs, `Axiom/docs/**`, and `Axiom/packages/tracker*/src/*.ts`
  before being written.

## Acceptance criteria

- [x] All 13 files exist under `Axiom.Spec/specs/manuales/`.
- [x] Every relative cross-link written (sibling `manuales/*.md` links, and
      `../08_Glosario.md` / `../05_Interfaces_Operativas.md`) resolves to a
      file that exists.
- [x] The Azure DevOps plugin's config file (`tracker.json`) and API-key/PAT
      location (env var named by `patEnvVar`, else Windows user-scoped env
      var, else `SecretStore` key `axiom.ado.<org>.<project>.pat`, else an
      interactive prompt that persists to the secret store) are documented
      accurately.
- [x] The launcher visual front, its pre-launch doctor gate, the onboarding
      tab (install/join/roles), and per-adapter agent tuning are documented
      accurately (grounded in the four `INC-20260715-*` increment specs).
- [x] `Axiom.Spec/specs/README.md` links to the manuals index.

## Open questions

none — resolved by orchestrator

## Assumptions

- The manuals are written for THIS workspace's installed configuration
  (Axiom.SDD + Axiom.Spec + Axiom, with the Azure DevOps tracker as the one
  installed optional plugin) — not a generic multi-plugin product manual.
- Spanish voice matching the rest of `Axiom.Spec/specs/`, with identifiers/
  commands kept in English/code, consistent with `05_Interfaces_Operativas.md`
  and `08_Glosario.md`.

## Implementation notes

Read before writing (for accuracy and voice): `Axiom.Spec/specs/README.md`,
`05_Interfaces_Operativas.md`, `08_Glosario.md`, `04_Flujos_SDD_y_Ciclo_de_Vida.md`,
the four `INC-20260715-adapter-agent-tuning` / `INC-20260715-launcher-ado-bridge`
/ `INC-20260715-launcher-doctor-gate` / `INC-20260715-launcher-onboarding`
increment specs, `Axiom/docs/{installation,cli/*,configuration/*}.md`, and the
tracker config/auth source (`Axiom/packages/tracker/src/config.ts`,
`Axiom/packages/tracker-ado/src/ado-tracker.ts`,
`Axiom/packages/tracker-ado/src/user-scoped-env.ts`) to confirm the exact
`tracker.json` shape and PAT-resolution order before documenting them.

Files created under `Axiom.Spec/specs/manuales/`:
`README.md`, `01_Que_Es_Cada_Cosa.md`, `02_Configuracion.md`,
`03_Actualizar_Versiones.md`, `04_Generar_Spec_y_Contexto_Tecnico.md`,
`05_Incrementos.md`, `06_Bugs.md`, `07_Planes.md`, `08_Implementacion.md`,
`09_Revisiones.md`, `10_Archivado.md`, `11_Launcher_Visual.md`,
`12_Plugin_Azure_DevOps.md`.

`Axiom.Spec/specs/README.md` edited: added one `manuales/` line under
"Artefactos específicos" pointing to `manuales/README.md`.

## Validation

No validation command was found. Performed best-effort validation by
inspecting changed files and checking consistency against the requested
behavior. Specifically:

- Listed `Axiom.Spec/specs/manuales/` and confirmed all 13 files are present.
- Manually walked every relative link written in each of the 13 files
  (`NN_Name.md` sibling links, `README.md` links, and the two `../` links to
  `05_Interfaces_Operativas.md` and `08_Glosario.md`) and confirmed each
  target file exists on disk.
- Confirmed `git status` in `Axiom/` (product repo) shows no changes
  introduced by this increment (docs-only work happened entirely in
  `Axiom.Spec/`).
- Cross-checked every command/config-field claim (tracker.json shape, PAT
  resolution order, launcher endpoints, doctor gate, onboarding tab, agent
  tuning defaults, increment/bug/plan/role lifecycle verbs) against the
  source files listed in `## Implementation notes` before writing.

## Result

Created the `manuales/` section: an index plus 12 substantive how-to pages
covering the product model, configuration, upgrades, spec/technical-context
generation, and the increment/bug/plan/implementation/review/archive
lifecycle, plus two pages specific to this installation's peculiarities (the
browser-based launcher visual front and the Azure DevOps tracker plugin,
including exactly what to fill in when creating increments/bugs and where the
PAT/API key lives). Added one navigation entry to `specs/README.md`. No
product code was touched; no existing 00–08 spec file was edited.

## General spec integration

Integrado por el pase final de la orquestación (autopilot, 2026-07-15):
- `README.md` (índice de specs) → entrada de navegación `manuales/` (ya añadida por este incremento).
- `01_Requisitos_Funcionales.md` → **RF-AXM-033 Manuales de operación en la spec**.
- `00_Resumen_Ejecutivo.md` → mención de `specs/manuales/` en la tanda.
- `08_Glosario.md` → término "Manuales de Axiom (`specs/manuales/`)".
- Cross-referenciado desde `04`/`05`/`06`/`07` (enlaces a las páginas de manuales correspondientes).

Archivado en `Axiom.Spec/specs/increments/_archive/`.
