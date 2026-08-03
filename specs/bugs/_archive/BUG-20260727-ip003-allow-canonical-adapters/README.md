# Bug: IP-003 must gate on the canonical adapter registry, not on MVP_TARGETS — restore allowedTargets to all 10

Status: closed
Date: 2026-07-27

Supersedes the direction of `BUG-20260727-ip003-default-profiles-allowedtargets`
(closed 2026-07-27, `Axiom.Spec/specs/bugs/BUG-20260727-ip003-default-profiles-allowedtargets/README.md`).
That fix chose the wrong resolution for a real IP-003 problem: it
restricted `DEFAULT_PROFILES`' `allowedTargets` to the 5 `MVP_TARGETS`
instead of fixing IP-003's validation set. Its own "Known follow-up"
section explicitly documented the regression this caused and recommended
opening a dedicated follow-up bug — this is that bug.

## Symptom

After `BUG-20260727-ip003-default-profiles-allowedtargets` shipped,
`generateWorkspaceAdapters` (`apps/cli/src/commands/workspace-adapters.ts`)
silently stopped generating any files when the *primary* adapter
(`adapters[0]`) was one of the 5 non-MVP targets
(`cursor`/`github-copilot`/`litellm`/`vscode`/`codex`). Confirmed by 3
pre-existing tests failing in `apps/cli/tests/workspace-adapters.test.ts`
("cursor/github-copilot/litellm...", "vscode escribe...", "codex escribe
un AGENTS.md real...").

## Current behavior (before this fix)

- `packages/capability-model/src/constants.ts`'s `MVP_TARGETS` (5 targets)
  was the only "adapter target" list doctor's IP-003
  (`packages/doctor/src/checks.ts`) validated `allowedTargets` against.
- `packages/install-profiles/src/default-profiles.ts`'s `DEFAULT_PROFILES`
  had both `profileBindings['product-owner'].allowedTargets` and
  `profileBindings['builder'].allowedTargets` hardcoded to `[...MVP_TARGETS]`
  (5 entries) instead of the full 10-target canonical registry
  (`apps/cli/src/commands/init.ts`'s `ADAPTER_TARGETS`).
- `packages/install-profiles/src/composer.ts`'s `resolveInstallProfile`
  (`validateTriple`) rejects any `adapterTarget` not present in
  `profileBindings[functionalProfile].allowedTargets`, throwing.
- `generateWorkspaceAdapters` calls `installProfile` with
  `adapterTarget: primaryAdapter (= adapters[0])` and
  `profilesData: DEFAULT_PROFILES` (in-memory, never persisted to disk).
  With `allowedTargets` restricted to the 5 MVP targets, selecting any of
  the 5 non-MVP targets as the *primary* adapter made `installProfile`
  return `composition-failed`, which `generateWorkspaceAdapters` degrades
  into a "no files generated" warning for that repo — contradicting the
  explicit, already-shipped goal (INC-20260726-adapter-registry-canonical)
  that all 10 canonical adapters are first-class and usable.
- The curated `Axiom/axiom.config/profiles.yaml` mirrored the same
  5-target restriction, with a comment explicitly attributing it to
  "IP-003 exige... 5 targets MVP".

## Expected behavior

1. An install profile MAY allow any of the 10 canonical adapter targets
   in `allowedTargets`, so `generateWorkspaceAdapters` can generate any of
   them (as primary or additional adapter) and adoption/setup with any of
   the 10 succeeds.
2. Doctor's **IP-003** ("Targets desconocidos") FAILS only when
   `allowedTargets` contains a target NOT in the canonical registry (a
   typo/unknown id) — NOT when it contains a known-but-non-MVP target. It
   validates `allowedTargets ⊆ CANONICAL_ADAPTER_TARGETS` (10 targets),
   not `⊆ MVP_TARGETS` (5 targets).
3. `MVP_TARGETS` and anything that legitimately depends on it for its
   OWN concept (model-routing/gateway support-matrix tiering, e.g. CC-006
   / `packages/model-routing/src/support-matrix.ts`) stays unchanged — MVP
   is a separate concept, not an install-profile allow-list gate.

## Impact

Before this fix: a project could not be adopted/set up with `cursor`,
`github-copilot`, `litellm`, `vscode`, or `codex` as its primary adapter
via `generateWorkspaceAdapters` — a real functional regression on top of
an already-shipped "all 10 adapters are first-class" increment
(INC-20260726-adapter-registry-canonical). Fixed: all 10 succeed as
primary or additional adapters again, while IP-003 still catches genuine
typos/unknown target ids.

## Reproduction steps (of the regression, now fixed)

1. Call `generateWorkspaceAdapters` with `adapters: ['cursor', ...]` (or
   `github-copilot`/`litellm`/`vscode`/`codex` as `adapters[0]`) against a
   repo with no project-level `axiom.config/profiles.yaml` override.
2. Before the fix: `installProfile` returns `composition-failed`
   (`adapterTarget "cursor" is not in profileBindings.builder.
   allowedTargets`), `generateWorkspaceAdapters` emits a warning, and no
   files are created for that repo.
3. After the fix: files are created as expected.

## Suspected cause (confirmed)

`DEFAULT_PROFILES.profileBindings[*].allowedTargets` was restricted to
`MVP_TARGETS` (a model-routing/gateway-support concept) instead of the
full canonical adapter registry, and doctor's IP-003 validated against the
same wrong (too-narrow) list — so "fixing" `DEFAULT_PROFILES` to satisfy
IP-003 required narrowing the actually-correct allow-list instead of
correcting IP-003's validation set.

## Acceptance criteria

- [x] A single canonical adapter-target list source exists for doctor to
      import (`CANONICAL_ADAPTER_TARGETS`, `@axiom/capability-model`), the
      10 ids matching `apps/cli/src/commands/init.ts`'s `ADAPTER_TARGETS`.
- [x] Doctor's IP-003 validates `allowedTargets ⊆ CANONICAL_ADAPTER_TARGETS`
      (not `⊆ MVP_TARGETS`); "Targets desconocidos" now means genuinely
      unknown/typo ids.
- [x] `DEFAULT_PROFILES.profileBindings['product-owner'].allowedTargets`
      and `...['builder'].allowedTargets` are both the full canonical 10.
- [x] The curated `Axiom/axiom.config/profiles.yaml` mirrors the same
      (both bindings' `allowedTargets` = 10; stale MVP-5 comment updated).
- [x] `generateWorkspaceAdapters` generates output for all 10 targets,
      including as the primary adapter — the 3 previously-broken tests in
      `apps/cli/tests/workspace-adapters.test.ts` pass again.
- [x] `MVP_TARGETS` (`@axiom/capability-model`) and CC-006 (environment
      matrix, `packages/model-routing`) are unchanged.
- [x] A guard test asserts `init.ts`'s `ADAPTER_TARGETS`,
      `CANONICAL_ADAPTER_TARGETS`, and `DEFAULT_PROFILES.adapterTargets`'
      ids stay identical (fails loudly on drift, given full single-sourcing
      was judged unnecessarily heavy for this fix).
- [x] `npm run build` passes.
- [x] `npx vitest run packages/install-profiles packages/installer
      packages/doctor apps/cli/tests/workspace-adapters.test.ts
      apps/cli/tests/e2e/adapters.e2e.test.ts` passes.

## Fix notes

1. `Axiom/packages/capability-model/src/constants.ts`: added
   `CANONICAL_ADAPTER_TARGETS` (10 ids, mirrors `init.ts`'s
   `ADAPTER_TARGETS`) next to the unchanged `MVP_TARGETS` (5), with a
   doc comment clarifying `MVP_TARGETS` is a model-routing/support-matrix
   concept, not an install-profile allow-list gate. Exported from
   `packages/capability-model/src/index.ts`.
   - Not single-sourced from `apps/cli/src/commands/init.ts` directly
     (would create a dependency cycle: `apps/cli` already depends on
     `@axiom/capability-model`). Mirrored instead, with a guard test (see
     below) to catch drift — documented as the deliberate tradeoff.
2. `Axiom/packages/doctor/src/checks.ts`: IP-003 now imports
   `CANONICAL_ADAPTER_TARGETS` instead of `MVP_TARGETS` and validates
   `allowedTargets` against it. Removed the now-unused `MVP_TARGETS`
   import from this file (the constant itself is untouched in
   `@axiom/capability-model`).
3. `Axiom/packages/install-profiles/src/default-profiles.ts`: both
   `profileBindings['product-owner'].allowedTargets` and
   `...['builder'].allowedTargets` reverted from `[...MVP_TARGETS]` to
   `[...CANONICAL_ADAPTER_TARGETS]`. Import switched accordingly. Header
   comment rewritten to explain the two-tier contract and cross-reference
   both this bug and the superseded one.
4. `Axiom/axiom.config/profiles.yaml`: both bindings' `allowedTargets`
   extended to the 10 canonical ids; the stale "restringe a los 5
   MVP_TARGETS... IP-003 exige" comment replaced with one describing the
   corrected IP-003 semantics.
5. `generateWorkspaceAdapters` (`apps/cli/src/commands/
   workspace-adapters.ts`) required no code change — it was already
   correct; it only needed `DEFAULT_PROFILES` to allow all 10 targets
   again.
6. Tests updated:
   - `packages/install-profiles/tests/default-profiles.test.ts`: replaced
     the MVP-vs-non-MVP split assertions with "all 10 canonical targets
     succeed, for both profiles" + "an unknown/typo target is rejected" +
     an IP-003-shaped assertion (`allowedTargets ⊆ CANONICAL_ADAPTER_TARGETS`,
     and exactly equal to it as a set).
   - `packages/installer/tests/installer.test.ts`: replaced the "5 MVP
     succeed / 5 non-MVP rejected" pair with "all 10 succeed" +
     "an unknown/typo target is rejected", against the bundled
     `DEFAULT_PROFILES` fallback.
   - `packages/doctor/tests/install-profiles.test.ts`: no change needed —
     its `VALID_PROFILES_YAML` fixture only declares `opencode`/
     `copilot-vscode` in `allowedTargets`, both members of
     `CANONICAL_ADAPTER_TARGETS`, so IP-003 still passes; its "fails when
     an unknown target is in allowedTargets" test still fails correctly
     (the target is unknown under either list).
   - `apps/cli/tests/workspace-adapters.test.ts`: no change needed for the
     existing 8 tests (they pass once `DEFAULT_PROFILES` is reverted);
     added a new "Guard — canonical adapter target lists stay in sync"
     `describe` block (2 tests) asserting `init.ts`'s `ADAPTER_TARGETS`,
     `CANONICAL_ADAPTER_TARGETS`, and `DEFAULT_PROFILES.adapterTargets`
     ids are identical sets, and that `allowedTargets` stays a subset of
     `CANONICAL_ADAPTER_TARGETS`.

## Validation

From `Axiom/`:

- `npm run build` — passes, no TypeScript errors.
- `npx vitest run packages/install-profiles packages/installer
  packages/doctor apps/cli/tests/workspace-adapters.test.ts
  apps/cli/tests/e2e/adapters.e2e.test.ts` — 28 test files, 285 tests, all
  passing. `apps/cli/tests/workspace-adapters.test.ts` alone: 10 tests
  (8 pre-existing + 2 new guard tests), all passing — confirms the 3
  previously-broken tests (cursor/github-copilot/litellm dispatch, vscode
  dispatch, codex dispatch) pass again.
- `npx vitest run packages/model-routing packages/capability-model` — 15
  test files, 141 tests, all passing — confirms `MVP_TARGETS`/CC-006/
  support-matrix are unaffected.

## Result

Fixed. IP-003 now gates on the true canonical registry
(`CANONICAL_ADAPTER_TARGETS`, 10 targets) instead of the narrower
model-routing `MVP_TARGETS` (5 targets), so `DEFAULT_PROFILES` (and the
curated `Axiom/axiom.config/profiles.yaml`) can allow — and does allow —
all 10 adapters in `allowedTargets` without failing doctor. This restores
`generateWorkspaceAdapters`'s ability to generate output for any of the 10
targets as the primary adapter, closing the regression the prior fix
introduced and explicitly flagged as a follow-up. IP-003 still correctly
rejects genuine unknown/typo target ids.

## General spec integration

No edit made to `Axiom.Spec/specs/00..08` (out of scope per this task's
explicit instructions). Nothing new needs integrating there in substance:
`Axiom.Spec/specs/06_Integraciones_y_Capacidades.md` (§"Registro canónico
de 10 adapter targets", INC-20260726-adapter-registry-canonical) already
states the canonical/correct model — that `default-profiles.ts`'s
`allowedTargets` for both profiles is part of the single-source
reconciliation of the 10-target vocabulary. This fix restores code
behavior to match that already-canonical spec text; the earlier,
now-superseded bug had drifted code away from it. The one piece of new,
durable knowledge — "doctor's IP-003 must validate against the full
canonical adapter registry, not against the narrower model-routing
MVP_TARGETS, because the latter is a distinct support-matrix-tiering
concept" — is recorded here and in the `CANONICAL_ADAPTER_TARGETS` doc
comment (`packages/capability-model/src/constants.ts`) for a maintainer to
fold into `06_Integraciones_y_Capacidades.md` in a dedicated spec-repo
pass if desired.
