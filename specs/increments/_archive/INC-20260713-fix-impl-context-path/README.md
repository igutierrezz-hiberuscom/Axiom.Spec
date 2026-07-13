# INC-20260713 — Thread role-derived spec rel-path into `implementationContextRead` (FIX-D)

## Goal

Extend FIX-A's spec-scope path convergence into the flagship **composite** MCP handler
`buildImplementationContext` (`packages/mcp-tools/src/implementation-context-handler.ts`,
backing `spec.implementationContextRead`), so that on a **dedicated spec repo** (its
`axiom.yaml` `role === 'spec'`) the composite bundle returns a populated `plan`,
`relatedSpec`, and `allowedWriteScope` — the same tree the CLI lifecycle, the direct spec
MCP read builders, and migration already converge on after FIX-A.

## The precise bug (root cause)

FIX-A (INC-20260713-fix-spec-scope-path) made spec artifacts for a **dedicated** spec repo
live at the spec-scope ROOT (`<scope>/increments|plans|bugs`, **no** `axiom.spec` segment),
and threaded a role-derived `specRelPath` into the **direct** MCP read builders
(`packages/mcp-server/src/input-builders.ts` + `context.ts`) via the helper
`resolveSpecRelPathForScope(role)` in `@axiom/workflow`
(`role === 'spec' ? '.' : 'axiom.spec'`).

But the **composite** handler `buildImplementationContext` re-resolves the spec repo itself
(via `getProjectRepos` → `repositories.spec.path`) and calls the artifact-store reads
**without** a `specRelPath`:

- `getPlan({ projectRoot: specRepoRoot, id })`
- `getIncrement({ projectRoot: specRepoRoot, id: incrementId })` (the `relatedSpec` read)
- `getAdrIndex({ projectRoot: specRepoRoot })`
- `getAllowedWriteScope({ projectRoot: specRepoRoot, id })`
- plus the path/body helpers `resolveArtifactDir(...)` / `readArtifactBody(...)` for
  `plan.path`, `relatedSpec.path`, and large-budget ADR bodies.

All of these fell back to `DEFAULT_SPEC_REL_PATH = 'axiom.spec'` and looked under
`<scope>/axiom.spec/plans|increments|adr/…` — which does **not** exist for a dedicated spec
repo. Result: `spec.implementationContextRead` returned `plan: null`, `relatedSpec: null`,
`allowedWriteScope: []` (and a degraded `confidence`), even though `repoSkills` +
`technicalContext` populated correctly (they use different resolvers not keyed on the
`axiom.spec` segment). The implementer flow's core "WHAT" bundle was therefore empty on a
dedicated spec repo.

## Scope

- In `packages/mcp-tools/src/implementation-context-handler.ts`, derive the spec repo's
  artifact rel-path from the **resolved spec repo's role** and thread it into **every**
  artifact-store read that targets the spec scope:
  - Reuse FIX-A's single source of truth: import `resolveSpecRelPathForScope` from
    `@axiom/workflow`.
  - Derive the role exactly as FIX-A's `context.ts` does — `resolveProject(specRepoRoot)`
    (`@axiom/project-resolution`); a `resolved` status with `role === 'spec'` → `'.'`,
    everything else → `'axiom.spec'` (via the helper). Best-effort: any resolution failure
    leaves the default in place.
  - Pass the derived `specRelPath` into `getPlan`, `getIncrement` (relatedSpec),
    `getAdrIndex`, `getAllowedWriteScope`, and the `resolveArtifactDir` / `readArtifactBody`
    calls for `plan.path`, `relatedSpec.path`, and large-budget ADR bodies.
- Add `@axiom/project-resolution` to `@axiom/mcp-tools`'s `package.json` deps +
  `tsconfig.json` paths/references (new runtime dependency; no cycle).

## Non-goals

- No change to the technical-context or skills resolvers (they already populate correctly
  and are not keyed on the `axiom.spec` segment).
- No change to behavior for self-hosted / single-repo / co-located spec scopes: they keep
  `axiom.spec` — **byte-identical** (`resolveSpecRelPathForScope(undefined)` returns
  `'axiom.spec'`, which resolves to the same path as passing no `specRelPath`).
- No new copy of the `. vs axiom.spec` rule — reuse the existing helper.
- The subfolder-spec-scope-in-multi-repo case (spec scope != spec repo root) stays
  deferred, exactly as in FIX-A.

## Acceptance

- On a **dedicated** spec repo (`axiom.yaml` `role === 'spec'`) whose plan + increment live
  at `<scope>/plans|increments` (no `axiom.spec`) and whose plan declares
  `allowedWriteScope`, `buildImplementationContext` with **default** args returns a non-null
  `plan`, a non-null `relatedSpec` (the linked increment), and a non-empty
  `allowedWriteScope`; `confidence` improves accordingly.
- **Regression**: a self-hosted / single-repo project (no `axiom.yaml` `role: spec`) still
  reads from `<root>/axiom.spec/…` exactly as before — every pre-existing
  `implementation-context-handler` test stays green.
- Gate green: `npm run build` → `npm test` → `npm run typecheck` → `npm run doctor` in
  `Axiom/`.

## Result

**Done — green.** Implemented in `packages/mcp-tools/src/implementation-context-handler.ts`:

- Added a private `resolveSpecArtifactRelPath(specRepoRoot)` helper that derives the
  role via `resolveProject(specRepoRoot)` (`@axiom/project-resolution`) — a `resolved`
  status with `role === 'spec'` → the FIX-A helper `resolveSpecRelPathForScope(role)`
  returns `'.'`; every other case (including unresolvable) returns `'axiom.spec'`. No new
  copy of the rule — the `@axiom/workflow` helper stays the single source of truth. The
  derivation is best-effort (try/catch → default).
- Computed `specRelPath` once (right after `specRepoRoot` resolves) and threaded it into
  every spec-scope artifact-store read: `getPlan`, `getIncrement` (relatedSpec),
  `getAdrIndex`, `getAllowedWriteScope`, plus the `resolveArtifactDir` / `readArtifactBody`
  calls for `plan.path`, `relatedSpec.path`, and large-budget ADR bodies (`readArtifactBody`
  gained a `specRelPath?` param). The technical-context + skills reads use different
  resolvers (not keyed on `axiom.spec`) and were correctly left untouched.
- Added `@axiom/project-resolution` to `@axiom/mcp-tools`'s `package.json` deps +
  `tsconfig.json` paths/references (no dependency cycle).

**Why the registry role was NOT used**: `repositories.spec.role` is a denormalized copy of
the `repos` map key — always literally `'spec'` when selected via `repos['spec']` — so it
cannot discriminate a dedicated spec repo from a self-hosted one. `resolveProject().role`
(the `axiom.yaml` v2 `role` field) is the correct discriminator, matching FIX-A's
`context.ts` exactly.

**Tests** (`packages/mcp-tools/tests/implementation-context-handler.test.ts`, +3): a
dedicated-spec-repo fixture (`axiom.yaml` `role: spec`, artifacts at the scope root) proves
`buildImplementationContext` with default args now returns non-null `plan` (path at the
root, no `axiom.spec`), non-null `relatedSpec`, and non-empty `allowedWriteScope`, with
content inlining at `medium`; a regression case (`axiom.yaml` `role: code`, artifacts under
`axiom.spec`) proves the self-hosted default is byte-identical. All 15 pre-existing handler
tests stay green.

**Gate**: `npm run build`, `npm test` (274 files / 2782 tests, +3 vs 2779 baseline, 0
failures), `npm run typecheck`, and `node apps/cli/dist/index.js doctor` (`Resultado: PASS`,
0 failures) all green.
