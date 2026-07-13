# INC-20260713 — Unify spec-scope path resolution (FIX-A)

## Goal

Make **CLI lifecycle commands**, the **spec MCP broker**, and **migration/adoption**
resolve spec artifacts (`increments/`, `bugs/`, `plans/`, `adr/`, `decisions/`) at the
**same on-disk tree** for a project. Today they diverge for a **dedicated spec repo**:
migration writes artifacts directly under the resolved spec scope, but the CLI lifecycle
and the spec MCP broker force-append an `axiom.spec` segment (the `DEFAULT_SPEC_REL_PATH`
in `packages/workflow/src/artifact-store.ts`). Result: for a dedicated spec repo (e.g.
KVP25 `Kvp.Spec/specs`), CLI-created and migrated artifacts land in two different trees;
`axiom-increment list` and `spec.incrementRead`/`planRead` miss the migrated ones.

## Owner decision (the fix)

The `axiom.spec` segment must **not** be force-appended for a dedicated spec repo — the
spec scope **is** the path passed, and artifacts live **directly** under it
(`<scope>/increments`). The `axiom.spec` subfolder only makes sense for the self-hosted /
co-located case (specs sharing a code repo, e.g. the Axiom product repo's own
`Axiom/axiom.spec`).

Therefore: **derive the spec artifact rel-path from the resolved spec repo's ROLE**:

- Resolved spec repo is a **dedicated spec repo** (`axiom.yaml` `role === 'spec'`, i.e. a
  multi-repo topology with a real spec repo) → rel-path = `.` (artifacts at the scope root).
- Otherwise (single-repo / self-hosted / co-located / `role !== 'spec'`) → rel-path =
  `axiom.spec` (**unchanged** — preserves the dogfood + all existing single-repo tests).

Then make migration, the CLI lifecycle, and the spec MCP broker all use this **one**
derived resolution so they converge on the same tree.

## Scope

- New pure helper `resolveSpecRelPathForScope(role)` in `@axiom/workflow` — the single
  source of truth for the `.` vs `axiom.spec` decision (exact check: `role === 'spec'`).
- New CLI shared module `apps/cli/src/commands/_spec-scope.ts`:
  `resolveSpecScopeAbsolutePath` (extracted from `bootstrap.ts`, so migration + CLI share
  it) and `resolveSpecArtifactRelPath(projectRoot)` (derives the concrete rel-path to
  thread to the artifact store, gated on `role === 'spec'`).
- Thread the derived rel-path through every CLI artifact-store reader/writer/list/archive
  call site: `axiom-increment`, `axiom-bug`, `axiom-plan`, `axiom-adr`, `axiom-decision`,
  `artifact-metadata-cli` (link / external-ref / **list**), `index-cmd`, `integrate`,
  `normalize-cmd`, `state-cmd`, `validate-changes`, `scaffold`, `_functional-verify`,
  `_role-review`, `app-launcher`.
- Spec MCP broker: derive the rel-path from the pinned project's spec repo role
  (`packages/mcp-server/src/context.ts`) and default `spec.*` read handlers to it
  (`packages/mcp-server/src/input-builders.ts`).

## Non-goals

- Changing the `DEFAULT_SPEC_REL_PATH` constant (stays `axiom.spec`, the self-hosted
  default).
- Changing single-repo / self-hosted / co-located behavior in any way (byte-identical).
- Changing migration's target for non-dedicated projects (it already resolves the real
  spec scope; unchanged).
- The subfolder-spec-scope-over-MCP case (Q-001, deferred).

## Path model (truth table)

| Resolved spec repo                     | rel-path    | Artifact tree                     |
|----------------------------------------|-------------|-----------------------------------|
| Dedicated spec repo (`role === 'spec'`) | `.`         | `<scope>/increments`, `<scope>/bugs`, … |
| Self-hosted / single-repo / co-located  | `axiom.spec`| `<scope>/axiom.spec/increments`, … |

`<scope>` = the resolved spec scope absolute path (`resolveSpecScopeAbsolutePath`), which
for a dedicated spec repo equals the spec repo root (topology `specRepo` path / the
`role: spec` repo whose `paths.self` is `.`). For a dedicated spec repo the concrete
rel-path threaded to the artifact store is `path.relative(projectRoot, scope)`
(normalised to `.` when `scope === projectRoot`), which equals migration's existing
computation — hence convergence.

## Acceptance

1. `resolveSpecRelPathForScope('spec')` → `.`; `resolveSpecRelPathForScope(anything-else /
   undefined)` → `axiom.spec`.
2. Dedicated spec repo (multi-repo, `role: spec`): `axiom-increment create` writes to
   `<scope>/increments` (no `axiom.spec`); `axiom-increment list` finds it; `spec.incrementRead`
   reads it with **default** MCP args (no explicit `specRelPath`).
3. A migrated artifact (from-legacy-sdd into the same scope) is co-located with the
   CLI-created one and is listed + MCP-readable → convergence proven.
4. Regression: single-repo / self-hosted create / list / read still resolve
   `<root>/axiom.spec/…` byte-identically.

## Result

**Done (archived).** All four gates green:

- `npm run build` — green.
- `npm test` — **2739 passed** (2731 baseline + 8 new; 0 regressions).
- `npm run typecheck` — green.
- `npm run doctor` — **PASS** (0 failures).

Delivered:

- `resolveSpecRelPathForScope(role)` in `@axiom/workflow` — the single role check
  (`role === 'spec' ? '.' : 'axiom.spec'`), unit-tested in `artifact-store.test.ts`.
- `resolveSpecArtifactRelPath(projectRoot)` + extracted `resolveSpecScopeAbsolutePath`
  in the new CLI `apps/cli/src/commands/_spec-scope.ts`; threaded through every CLI
  artifact-store reader/writer/list/archive call (increment, bug, plan, adr, decision,
  artifact-metadata-cli link/external-ref/list, index-cmd, integrate, normalize, state,
  validate-changes, scaffold, `_functional-verify`, `_role-review`, app-launcher). Base
  `projectRoot` is never changed — only the derived `specRelPath` is threaded — so
  single-repo / self-hosted stays byte-identical.
- Migration (`bootstrap.ts`) now shares the extracted `resolveSpecScopeAbsolutePath`; its
  existing target (`<scope>/increments`) already equals the CLI's derived target for a
  dedicated spec repo → convergence.
- Spec MCP broker: `context.specArtifactRelPath` is derived from the resolved spec repo's
  role (`packages/mcp-server/src/context.ts`) and the `spec.*` read builders default to it
  (`input-builders.ts`).

Convergence proven by `apps/cli/tests/spec-scope-convergence.test.ts` (CLI-created +
migrated increments co-located under `<scope>/increments`, both listed) and
`packages/mcp-server/tests/spec-scope-convergence.test.ts` (`spec.incrementRead` reads the
same tree with DEFAULT args), with single-repo regressions in both.
