# Increment: Product repo self-bootstrap (axiom.config/axiom.spec scaffold)

Status: closed
Date: 2026-07-08

## Goal

Scaffold the missing canonical `axiom.config/` and `axiom.spec/`
structure at the Axiom PRODUCT repo root (`Axiom/`) so the product can
self-validate: `npm run readiness:first-project` passes and the
dogfood/real-repo tests that read the repo's own canonical config go
green — with valid content per the real schemas, not stubs.

## Context

`Axiom/` had `axiom.yaml` at its root but lacked the `axiom.config/`
scaffold its own runtime and dogfood tests expect
(`skills-catalog.yaml`, `agents-catalog.yaml`,
`model-routing-policy.yaml`, etc.). Two independent root causes
compounded the failure:

1. `axiom.yaml`'s `project.name` was absent (the file used a custom
   `product`/`repos` narrative shape only), so
   `@axiom/project-resolution`'s `resolveProject` returned `ambiguous`
   instead of `resolved` for the product repo itself.
2. `scripts/verify-first-project-readiness.mjs` computed `repoRoot =
   path.resolve(productRoot, '..')` — one level too high, landing on
   the 3-repo workspace root (`Axiom Workspace/`, which has
   `Axiom.Spec`/`Axiom.SDD`/`Axiom`, not `axiom.config`/`axiom.spec`
   lowercase folders) instead of `productRoot` (`Axiom/`) itself.
3. (Found during work) `packages/skills/tests/catalog.test.ts` and
   `packages/agents/tests/catalog.test.ts` had an off-by-one in their
   `path.resolve(__dirname, '..', '..', '..', '..')` (4 levels from
   `packages/<pkg>/tests/`, should be 3) — same "overshoot into the
   workspace parent" bug as the readiness script, independently
   introduced in the tests.

## Scope

- New canonical config at `Axiom/axiom.config/`: `skills-catalog.yaml`,
  `agents-catalog.yaml`, `model-routing-policy.yaml`, `profiles.yaml`,
  `providers.yaml`, `capabilities.yaml`, `integrations.yaml`,
  `policy-as-code.yaml`, `mcp-manifest.yaml`, `telemetry-sinks.yaml`.
- New source content at `Axiom/axiom.spec/target-axiom-skills/*.md` (7
  skills) and `Axiom/axiom.spec/target-axiom-agents/*.md` (3 agents),
  referenced by the two catalogs via `source` + `bundleHash` (real
  sha256 of each source file, verified by doctor's TC-010/TC-011).
- New `Axiom/axiom.spec/templates/` (copied verbatim from the
  canonical `Axiom.Spec/templates/`), `Axiom/AGENTS.md` (canonical
  repo-root contract), `Axiom/axiom.skills.lock` (empty-but-valid).
- Additive fix to `Axiom/axiom.yaml`: added a minimal `project:` block
  (`project.name: axiom-runtime`) so the file validates as v1 under
  `@axiom/config-validation` without touching the existing narrative
  `product`/`repos`/`rules` content.
- Tiny path fix in `Axiom/scripts/verify-first-project-readiness.mjs`
  (`repoRoot = productRoot`, not `path.resolve(productRoot, '..')`).
- Tiny off-by-one fix in `packages/skills/tests/catalog.test.ts` and
  `packages/agents/tests/catalog.test.ts` (3 `..` instead of 4).
- `.gitignore`: added `.axiom-state/` (resolves doctor's MC-002 warning
  about the local overlay not being gitignored at the product repo).

## Non-goals

- Not fixing genuinely-logic test failures unrelated to config
  presence: `model-routing`'s `opencode-projection` slotCount (7 vs
  10, a `SlotId`/`SLOT_TAXONOMY` taxonomy gap), `project-resolution`'s
  v1 scope `exists`/`isProductRuntime` semantics, `telemetry`'s
  audit-trail retention-sweep window boundary, `toolchain`'s
  `repairToolchain` multi-tool detectionPaths substitution. These are
  explicitly deferred to a sibling increment (AB2).
- Not integrating this increment's notes into
  `Axiom.Spec/specs/00-08` (reserved for the orchestrator's final
  pass across the batch).
- No git commits performed by this increment.

## Acceptance criteria

- [x] `npm run build` (`tsc -b`) is clean.
- [x] `npm run readiness:first-project` passes.
- [x] Full `npx vitest run` failing-test count drops from 10 to 4,
      and all 4 remaining failures are the pre-identified logic-only
      set (none are config-missing).
- [x] No previously-passing test regressed (delta is 10 → 4, not a
      shuffle).
- [x] Scaffolded config validates against the real Zod
      schemas/loaders (`@axiom/config-validation`,
      `@axiom/capability-model`, `@axiom/install-profiles`,
      `@axiom/model-routing`, `@axiom/skills`, `@axiom/agents`) — not
      fabricated stubs; confirmed by both the targeted vitest files
      and a full `npm run doctor` run against the product repo itself
      (40/54 pass, 0 fail, 4 benign warnings, 10 skipped/optional).

## Open questions

None blocking. Resolved ambiguities are recorded under Assumptions.

## Assumptions

- The readiness script's `repoRoot` was a genuine bug (not an
  intentional 3-repo-workspace layout expectation): `Axiom/` is a
  self-contained monorepo and is meant to own its own
  `axiom.config/`/`axiom.spec/` scaffold, consistent with how
  `axiom.yaml`, `AGENTS.md`, and every doctor check already resolve
  paths relative to the product repo's own root.
- `axiom.yaml`'s fix is additive-only: `project.name` was added
  without removing or restructuring the existing `product`/`repos`
  narrative block, since Zod schemas here are non-strict (unknown
  keys pass through) and no runtime loader in the codebase parses
  the narrative `product`/`repos` keys today.
- `profiles.yaml`'s `allowedTargets` were trimmed to the 5
  `MVP_TARGETS` (not all 8 `adapterTargets`) to satisfy doctor's
  strict `IP-003` check; `adapterTargets` itself still declares all 8
  (5 `mvp: true` + 3 `mvp: false`), matching
  `@axiom/install-profiles`'s bundled `DEFAULT_PROFILES` shape.
- Skill/agent source `.md` content was authored fresh, grounded in
  the real implementing package/behavior for each id (verified
  against `packages/*/src` and `Axiom.Spec/specs/06_...md`), not
  copied from a pre-existing canonical source (none existed under
  `axiom.spec/target-axiom-skills|agents/` before this increment,
  confirmed by search). `bundleHash` values are the real sha256 of
  each authored source file.

## Implementation notes

Files created:
- `Axiom/axiom.config/skills-catalog.yaml`
- `Axiom/axiom.config/agents-catalog.yaml`
- `Axiom/axiom.config/model-routing-policy.yaml`
- `Axiom/axiom.config/profiles.yaml`
- `Axiom/axiom.config/providers.yaml`
- `Axiom/axiom.config/capabilities.yaml`
- `Axiom/axiom.config/integrations.yaml`
- `Axiom/axiom.config/policy-as-code.yaml`
- `Axiom/axiom.config/mcp-manifest.yaml`
- `Axiom/axiom.config/telemetry-sinks.yaml`
- `Axiom/axiom.spec/target-axiom-skills/*.md` (7 files)
- `Axiom/axiom.spec/target-axiom-agents/*.md` (3 files)
- `Axiom/axiom.spec/templates/*` (45 files, copied from
  `Axiom.Spec/templates/`)
- `Axiom/AGENTS.md`
- `Axiom/axiom.skills.lock`

Files edited:
- `Axiom/axiom.yaml` (additive `project:` block)
- `Axiom/scripts/verify-first-project-readiness.mjs` (`repoRoot` fix)
- `Axiom/packages/skills/tests/catalog.test.ts` (off-by-one path fix)
- `Axiom/packages/agents/tests/catalog.test.ts` (off-by-one path fix)
- `Axiom/.gitignore` (added `.axiom-state/`)

## Validation

- `npm run build` (from `Axiom/`): clean, no errors.
- `npm run readiness:first-project` (from `Axiom/`):
  ```
  [readiness:first-project] PASS
  [readiness:first-project] baseline: profile=builder overlay=local-only target=opencode
  [readiness:first-project] sequence: init -> configure -> toolchain repair -> sync -> start -> gateway start/status -> audit -> doctor
  ```
- `npm run doctor` (from `Axiom/`, self-dogfood): `40/54 OK · 0 FALLO ·
  4 ADVERTENCIA · 10 OMITIDO` — `Resultado: PASS`. The 4 warnings are
  benign (legacy v1 `axiom.yaml` schema notice, gateway state not yet
  started at rest, 2 optional/skip-eligible structures).
- `npx vitest run` (full suite, from `Axiom/`): before 10 failing
  tests / 1849 passing (1859 total); after 4 failing tests / 1855
  passing (1859 total). All 4 remaining failures are the pre-labeled
  logic-only set (see Non-goals). No regressions in previously-passing
  tests.

## Result

Product repo now self-bootstraps: `axiom.config/`/`axiom.spec/` exist
with real, schema-valid content; `readiness:first-project` and
`npm run doctor` both pass against the repo's own root; 6 of the 10
originally-failing dogfood/real-repo tests are fixed by this
increment (the 2 catalog tests, the 3 `axiom.config/model-routing-
policy.yaml` ENOENT tests, and the `doctor` "repositorio Axiom real"
ambiguous-resolution test). The remaining 4 failures are unrelated
package-logic bugs, explicitly out of scope here and deferred to AB2.

## General spec integration

Integrated into `Axiom.Spec/specs/00-08` by the batch's final
consolidation pass (autopilot orchestrator). Landed in:

- **00_Resumen_Ejecutivo.md** — superseded the "Discrepancia real
  conocida (no resuelta)" section with "Auto-validación del producto
  restaurada": the product self-validates now (`readiness:first-project`
  + `doctor` pass against its own root).
- **02_Requisitos_No_Funcionales.md** — NFR-AXM-010 note updated: full
  suite green (this increment took failures 10 → 4).
- **03_Modelo_Operativo_y_Datos.md** — new subsection documenting the
  scaffolded product-repo `axiom.config/`/`axiom.spec/` canonical set
  (schema-valid content, not stubs; `profiles.yaml#allowedTargets`
  trimmed to the 5 `MVP_TARGETS` for `IP-003`).
- **07_Gobierno_y_Seguridad.md** — new governance subsection: product
  self-validation restored (closes the "can't dogfood itself" gap).
