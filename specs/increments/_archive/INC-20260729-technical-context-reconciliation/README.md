# Increment: Reconcile technical context with the current Axiom runtime

Status: closed
Date: 2026-07-29

## Goal

Bring `Axiom.Spec/context/**` back into alignment with the current
implementation in `Axiom/`, without turning the technical context into a
code dump or presenting historical baselines as current state.

## Context

The technical-context tree was baselined on 2026-07-02. Several shipped
increments changed the runtime after that date: configuration and state
folder renames, config deduplication, the generic `init` wizard, adoption
scaffolding, the launcher default, additional adapters, MCP packages, the
`cmm` toolchain replacement, and expanded doctor checks. The entry point had
already been partially updated, but several linked documents still carried
contradictory baseline claims.

## Scope

- Reconcile `context/TECHNICAL_CONTEXT.md` and its index/date metadata.
- Reconcile the architecture documents for the current package inventory,
  configuration paths, lifecycle commands, onboarding flow, and launcher.
- Reconcile the operations documents for adoption scaffolding and current
  doctor/governance checks.
- Reconcile the integrations document for the current toolchain and native
  MCP server packages.
- Reconcile the reference documents for package/adapter counts, increment
  history, and known risks/breaches.
- Preserve historical details only when they are explicitly identified as
  historical or superseded.
- Record the stable workflow consequence in
  `Axiom.Spec/specs/04_Flujos_SDD_y_Ciclo_de_Vida.md`: the autopilot final
  pass also reconciles `context/**` before archiving.

## Non-goals

- No product-code changes in `Axiom/`.
- No redesign of the technical-context structure.
- No speculative architecture or new indexes, metadata systems, or runtime
  integrations.
- No removal of historical increment records merely because their original
  paths or counts are obsolete.

## Acceptance criteria

- [x] `TECHNICAL_CONTEXT.md` identifies 43 packages, 81 CLI command files,
      9 adapter packages, and the current `.axiom-state`/`axiom.config`
      paths with verifiable sources.
- [x] Current-state claims about `axiom.spec/`, MCP packages, `cmm`, the
      launcher, adoption scaffolding, and doctor checks match the real tree
      and cited source files.
- [x] No linked context document presents the old 28-package, 36-command,
      old-folder, or no-MCP baseline as current state.
- [x] Historical references retain an explicit historical/superseded label.
- [x] The context entry point has an updated validation date and an accurate
      document index.
- [x] The autopilot lifecycle spec records the new technical-context pass.
- [x] The reconciled context and all claims are independently reviewed
      against the current `Axiom/` tree.

## Open questions

None blocking.

## Assumptions

1. The current source of truth is the checked-out `Axiom/` tree, not stale
   generated `dist/` output or the 2026-07-02 context baseline.
2. Counts are recorded with their scope: 34 top-level packages plus 9
   adapter sub-packages, 81 command `*.ts` files, and 9 adapter packages.
3. Historical documents may mention old names and counts when they describe
   the state of an earlier increment, provided the surrounding text labels
   them as historical and does not use them as current runtime facts.

## Implementation notes

The reconciliation verifies the current state through the cited code and
repository files, including:

- `packages/filesystem-truth/src/discovery.ts` for
  `LOCAL_OVERLAY_DIRNAME = '.axiom-state'` and
  `AXIOM_CONFIG_DIRNAME = 'axiom.config'`.
- `packages/*/package.json`, `packages/adapters/*/package.json`, and
  `apps/cli/src/commands/*.ts` for the package, adapter, and command counts.
- `packages/mcp-server/src/` and `packages/mcp-tools/src/` for the native
  MCP server/handler surface.
- `packages/toolchain/src/probe.ts` and
  `docs/0031-adr-cmm-replaces-graphify-and-codegraph.md` for the current
  `cmm` replacement.
- `apps/cli/src/commands/workspace-setup.ts`,
  `workspace-adopt.ts`, `workspace-config-scaffold.ts`, and
  `workspace-catalog-scaffold.ts` for adoption scaffolding.
- `packages/doctor/src/` and `apps/cli/src/commands/app*.ts` for the
  current doctor and launcher behavior.

The context docs are updated in place and cite these sources. The risk
reference keeps its historical baseline material only as labeled history;
its resolution snapshot reflects the current product.

## Validation

Validation completed after the document corrections and spec integration:

- direct count/path checks against `Axiom/` confirmed 34 top-level packages,
  9 adapter packages, 81 command files, the `.axiom-state` and
  `axiom.config` paths, and the current MCP/workspace sources;
- targeted search found old names only in explicitly historical baseline
  sections or as the documented `_builder/` gap;
- every changed context document and the autopilot lifecycle subsection was
  reviewed after editing;
- `npm run build` pasó correctamente (`tsc -b`);
- `npx vitest run` terminó con código 1: pasaron 316 de 326 ficheros de
  prueba y 3329 de 3383 tests; quedaron 54 fallos en 10 ficheros. Un caso
  fue un timeout de 5000 ms en
  `apps/cli/tests/e2e/workspace-adopt.e2e.test.ts`; los demás se concentran
  en flujos de configuración, adopción, workspace, launcher y frescura Git
  del runtime. No son atribuibles a esta reconciliación documental, que no
  modificó `Axiom/`; no se afirma aquí que sean preexistentes sin una
  ejecución baseline independiente.

## Result

Implemented. The context entry point, architecture, operations,
integrations, and reference documents now describe the verified current
runtime and identify historical baselines as historical.

## General spec integration

Integrated into `Axiom.Spec/specs/04_Flujos_SDD_y_Ciclo_de_Vida.md`.
Updated context files: `TECHNICAL_CONTEXT.md`,
`architecture/01-vision-general-y-capas.md`,
`architecture/02-modelo-de-datos-y-configuracion.md`,
`architecture/03-ciclo-de-vida-cli-y-orquestacion.md`,
`operations/01-instalacion-y-onboarding.md`,
`operations/02-doctor-troubleshooting-y-telemetria.md`,
`integrations/01-capabilities-providers-y-toolchain.md`,
`references/01-inventario-de-packages.md`,
`references/02-historial-de-incrementos.md`, and
`references/03-riesgos-y-brechas-conocidas.md`. `architecture/04` was
reviewed and retained because it was already current; no new document was
added.
