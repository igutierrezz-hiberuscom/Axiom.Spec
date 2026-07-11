# INC-20260711-audit-bug-fixes — Fix confirmed audit bugs (correctness + consistency)

## Goal

Fix seven confirmed bugs surfaced by an audit of the Axiom monorepo. Each is a
correctness or consistency defect in already-shipped functionality; none is a new
feature. Diffs are minimal and follow existing patterns. The build and the full
test suite (2268 tests) MUST stay green, with new corner-case tests added where
flagged.

## Scope (7 items)

1. **Technical-context role-key fallback** — `packages/mcp-tools/src/implementation-context-handler.ts`.
   When the technical-context index lookup for the plan's `role` returns `null`,
   retry once with the generator default key (`DEFAULT_TECHNICAL_CONTEXT_ROLE` =
   `'repo'`) before declaring the technical context missing. So a project that
   only generated `repo.index.yml` still serves technical context to a
   `backend`-role plan.

2. **`from-legacy-sdd` writes to the resolved spec scope** — `apps/cli/src/commands/bootstrap.ts`
   + `apps/cli/src/bootstrap-from-legacy-sdd/migrator.ts`. `runBootstrapFromLegacySdd`
   migrated artifacts under the control repo's `axiom.spec/` default instead of the
   project's actual resolved spec scope. Reuse `resolveSpecScopeAbsolutePath`, error
   out on null like `from-code`, compute `specRelPath = relative(rootPath, specScope)`,
   and thread it through `migrateLegacyItems` to `artifactExists`/`saveArtifactMetadata`/
   `resolveArtifactDir`. `specRelPath` stays optional (defaults to `axiom.spec`).

3. **`axiom init --layout installed-multi-repo` no longer emits an unbacked sibling
   spec ref silently** — `apps/cli/src/commands/init.ts`. The emitted
   `specification: ../<name>.spec` now carries a `created: false` honesty marker,
   and the printed guidance to run `axiom workspace setup` is made prominent, so the
   user is not told about a spec repo that does not yet exist. `init` still does NOT
   create sibling repos. `packages/topology/src/loader.ts`'s
   `defaultInstalledMultiRepoManifest` is annotated for consistency (runtime
   behavior unchanged).

4. **`repo add` wires topology on a bare `init` project** — `apps/cli/src/commands/workspace-incremental.ts`.
   `runRepoAdd` previously warned and skipped `topology.yaml` when a control repo
   existed but no spec repo (the bare-init case). Now it synthesizes a self-hosted
   topology anchored at control (spec ref `.`, mirroring `defaultSingleRepoManifest`)
   that includes the newly added role repo, and writes `topology.yaml`.

5. **Windows 8.3 short/long path canonicalization** — new exported helper in
   `packages/filesystem-truth/src/`. Canonicalizes a path via
   `fs.realpathSync.native(p)` when it exists, falling back to `path.resolve(p)` +
   drive-letter normalization when it does not. Routed through the registry
   comparison sites (`findByRootPath`, `findByRepoPathV2`, `findByAncestorRepoPathV2`),
   `project-resolution`'s `normalizeForAncestorCompare`, and `workspace-setup`'s
   `relativeRef` inputs, so the same path in short vs. long form compares equal.

6. **`toolchain validate` warns for declared-but-absent non-mvp tools** —
   `packages/toolchain/src/validate.ts` + `types.ts`. The all-declared-tools loop
   only checked drift. It now emits a WARNING (never an error — exit-code semantics
   preserved; errors stay reserved for required/mvp tools) for any declared
   non-required tool whose detection state is `absent` (`optional-tool-absent`), and
   `marker`/`declared` when real detections are supplied (`optional-tool-unverified`).

7. **Cosmetic `-sdd-sdd` repoId double segment** — `apps/cli/src/commands/workspace-setup.ts`.
   `buildRoleAwareAxiomYaml` built `repoId = ${projectId}-${repoRole}-${roleKey}`,
   producing `<project>-sdd-sdd` / `<project>-spec-spec`. Collapse to a single
   segment when `repoRole === roleKey`. No lookup keys off `repoId` (they use
   `roleKey`/`topologyId`), so runtime impact is nil.

## Non-goals

- No behavior changes beyond these seven fixes.
- No new features.
- `init` does not create sibling repos (item 3); that remains `workspace setup`'s job.
- Item 5 does NOT rewrite stored registry path values at write time (see Decision
  D-001) — comparison-site canonicalization is sufficient to fix the bug and avoids
  breaking exact-path test assertions.

## Acceptance criteria

- Each of the 7 items implemented as described, with minimal diffs following
  existing patterns.
- `npm run build` succeeds in `Axiom/`.
- `npm test` (full suite) ends green.
- New corner-case tests added where flagged (items 1, 2, 4, 5, 6).
- Test assertions changed only where the intended behavior legitimately changed
  (items 1, 6, 7 were flagged as candidates); no tests weakened or deleted.

## Result

Status: **done**. `npm run build` (tsc -b) succeeds; full test suite green at
**2275 passed / 0 failed** (213 files) — 2268 baseline + 7 new tests. No existing
test assertions needed changing (the flagged candidates in items 1/6/7 turned out
not to require assertion edits given the actual fixtures).

Per-item:

1. **DONE** — `packages/mcp-tools/src/implementation-context-handler.ts`:
   imported `DEFAULT_TECHNICAL_CONTEXT_ROLE`; added the single-retry fallback to
   `'repo'`. New tests in
   `packages/mcp-tools/tests/implementation-context-handler.test.ts` (fallback
   finds `repo.index.yml` for a `backend` plan; still-missing when neither index
   exists). The pre-existing partial-data test was NOT flipped (it has no index
   at all, so the fallback also misses — assertion stayed valid).

2. **DONE** — `apps/cli/src/commands/bootstrap.ts` +
   `apps/cli/src/bootstrap-from-legacy-sdd/migrator.ts`: `runBootstrapFromLegacySdd`
   now resolves the spec scope (errors out on null like `from-code`) and threads
   `specRelPath` through `migrateLegacyItems` → `artifactExists`/
   `saveArtifactMetadata`/`resolveArtifactDir`. `specRelPath` optional (existing
   `migrator.test.ts` calls default to `axiom.spec`). New test in `bootstrap.test.ts`
   (spec scope `./spec` → artifacts under `./spec/...`, not `./axiom.spec/...`).

3. **DONE** — `apps/cli/src/commands/init.ts`: `buildAxiomYaml` installed-multi-repo
   `specification` entry now carries `created: false` + an honesty comment (zod
   strips the unknown key, so validation is unaffected); the printed guidance is
   now a prominent `⚠` banner instead of a buried `nota:`. `packages/topology/src/
   loader.ts`'s `defaultInstalledMultiRepoManifest` annotated for consistency
   (runtime unchanged).

4. **DONE** — `apps/cli/src/commands/workspace-incremental.ts`: `runRepoAdd`'s
   topology block now fires on `control !== undefined` (was `control && specRepo`);
   when no separate spec repo exists it synthesizes a self-hosted spec ref
   (`ref: '.'`, `mode: 'single-repo'`) that still includes the new role repo. New
   test in `workspace-incremental.test.ts` (`axiom init` + `repo add` → topology.yaml
   exists and includes the new role).

5. **DONE (compare-time; write-time intentionally omitted — see Decision D-001)** —
   new `canonicalizePath` in `packages/filesystem-truth/src/path-canonical.ts`
   (realpath when present; else canonicalize the deepest existing ancestor and
   re-append the missing tail — this partial-existence handling is required so a
   present path and a not-yet-created sibling canonicalize to consistent long
   forms). Routed through `registry.ts` (`findByRootPath`, `findByRepoPathV2`,
   `findByAncestorRepoPathV2`), `project-resolution`'s `normalizeForAncestorCompare`,
   and `workspace-setup`'s `relativeRef`. Added `@axiom/filesystem-truth` as a dep
   of `@axiom/user-workspace` (package.json + tsconfig references). New helper tests
   in `packages/filesystem-truth/tests/path-canonical.test.ts`. Stored registry
   paths are NOT rewritten at write time (would break unflagged exact-path
   assertions given the confirmed 8.3 short/long divergence on Windows); compare-time
   canonicalization of both operands fully fixes the cross-form matching bug.

6. **DONE** — `packages/toolchain/src/validate.ts` + `types.ts`: added
   `optional-tool-absent` (state `absent`) and `optional-tool-unverified`
   (`marker`/`declared`, only when real `detections` are supplied) WARNING finding
   kinds for declared-but-non-required tools; errors stay reserved for required/mvp
   tools (exit-code semantics unchanged). New test in `packages/toolchain/tests/
   toolchain.test.ts` (serena+rtk non-mvp, no detection paths → warnings, not a clean
   green). No existing toolchain/doctor test needed updating: TC-005 uses its own
   loop (not `validateToolchain`); TC-004 and the CLI `toHaveLength(2)` scenario have
   no declared-but-absent non-required tools.

7. **DONE** — `apps/cli/src/commands/workspace-setup.ts`: `buildRoleAwareAxiomYaml`
   collapses the `repoId` segment when `repoRole === roleKey` (no more
   `<project>-sdd-sdd` / `-spec-spec`). No test asserted the double segment, so no
   fixture change was required; nothing keys lookups off `repoId`.
