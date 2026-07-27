# Bug: doctor IP-003 fails on freshly-scaffolded projects (DEFAULT_PROFILES allowedTargets diverged from MVP_TARGETS)

Status: closed
Date: 2026-07-27

## Symptom

A freshly-scaffolded/adopted project fails `axiom doctor` check **IP-003**
with:

```
Targets desconocidos: product-owner:cursor, product-owner:github-copilot,
product-owner:litellm, product-owner:vscode, product-owner:codex,
builder:cursor, builder:github-copilot, builder:litellm, builder:vscode,
builder:codex
```

## Current behavior

Before the fix: `packages/install-profiles/src/default-profiles.ts`'s
`DEFAULT_PROFILES` listed all 10 adapter targets in BOTH profile bindings'
`allowedTargets` (`product-owner` and `builder`). `apps/cli/src/commands/
workspace-setup.ts` (line 705) writes `DEFAULT_PROFILES` verbatim (via
`yaml.dump`) to a project's `axiom.config/profiles.yaml` during workspace
scaffolding. Doctor's IP-003 (`packages/doctor/src/checks.ts` ~610-628)
then reads that same file and requires every `allowedTargets` entry to be a
member of `MVP_TARGETS` (`packages/capability-model/src/constants.ts`, the
5 canonical MVP targets: `copilot-vscode`, `opencode`, `antigravity`,
`claude-code`, `visual-studio-2026`). Since 5 of the 10 targets in
`allowedTargets` were not MVP targets, IP-003 failed on any project
scaffolded from the bundled default with no custom override.

The hand-curated `Axiom/axiom.config/profiles.yaml` (~105-128) already
restricted `allowedTargets` to the 5 MVP targets (with a comment
referencing IP-003) — it never had this bug. `DEFAULT_PROFILES` diverged
from it. INC-20260726-adapter-registry-canonical worsened the divergence by
adding `vscode`+`codex` to `DEFAULT_PROFILES`'s `allowedTargets` (both
profiles) without checking them against `MVP_TARGETS`.

## Expected behavior

A project scaffolded from `DEFAULT_PROFILES` must PASS IP-003: every
`profileBindings[*].allowedTargets` entry must be a member of
`MVP_TARGETS`. `adapterTargets` (the full 10-target selectable/declared
set) is unaffected — only the narrower per-binding `allowedTargets` is
restricted.

## Impact

Any project that ran `axiom init`/adopt and was scaffolded via
`workspace-setup.ts`'s `DEFAULT_PROFILES` dump (i.e. did not already have a
hand-curated `axiom.config/profiles.yaml`) failed `axiom doctor`'s IP-003
check out of the box, on a completely default, unmodified setup — a
first-run-broken developer experience regression.

## Reproduction steps

1. Scaffold/adopt a new project so that `axiom.config/profiles.yaml` gets
   written from `DEFAULT_PROFILES` (`apps/cli/src/commands/
   workspace-setup.ts`'s `materializeCanonicalProfilesYaml`-style dump, or
   any flow relying on the bundled fallback being persisted).
2. Run `axiom doctor` against that project.
3. Observe IP-003 `fail` with "Targets desconocidos" listing
   `cursor`/`github-copilot`/`litellm`/`vscode`/`codex` for both
   `product-owner` and `builder`.

## Suspected cause

Confirmed (not re-derived): `packages/install-profiles/src/
default-profiles.ts`'s `DEFAULT_PROFILES.profileBindings.{product-owner,
builder}.allowedTargets` listed all 10 `adapterTargets` ids instead of only
the 5 `MVP_TARGETS`, while `packages/doctor/src/checks.ts`'s IP-003 (and
`packages/capability-model/src/constants.ts`'s `MVP_TARGETS`) require
`allowedTargets ⊆ MVP_TARGETS`.

## Acceptance criteria

- [x] `DEFAULT_PROFILES`'s `profileBindings['product-owner'].allowedTargets`
      is exactly the 5 `MVP_TARGETS`.
- [x] `DEFAULT_PROFILES`'s `profileBindings['builder'].allowedTargets` is
      exactly the 5 `MVP_TARGETS`.
- [x] `DEFAULT_PROFILES.adapterTargets` is unchanged (still declares all
      10 CLI-selectable targets).
- [x] A regression test exists asserting every `profileBindings[*].
      allowedTargets` entry is a member of `MVP_TARGETS` (fails on revert).
- [x] `npm run build` passes.
- [x] `npx vitest run packages/install-profiles packages/doctor` passes
      (all IP-003 / default-profiles tests included).
- [x] The curated `Axiom/axiom.config/profiles.yaml` is reviewed for the
      same drift and reconciled where applicable.

## Fix notes

1. `Axiom/packages/install-profiles/src/default-profiles.ts`:
   - Added `import { MVP_TARGETS } from '@axiom/capability-model';`
     (no import cycle: `@axiom/install-profiles` already depends on
     `@axiom/capability-model`; the reverse dependency does not exist).
   - Changed both `profileBindings['product-owner'].allowedTargets` and
     `profileBindings['builder'].allowedTargets` from the hardcoded
     10-target array literal to `[...MVP_TARGETS]`, so this cannot drift
     out of sync with IP-003 again.
   - `adapterTargets` (10 entries, `mvp: false` for the 5 non-MVP ones)
     was left unchanged, per the explicit fix scope.
   - Updated the file's header comment to document the two-tier contract
     (`adapterTargets` = full declared/selectable set; `allowedTargets` =
     MVP-restricted subset enforced by IP-003) and cross-reference this
     bug.

2. `Axiom/axiom.config/profiles.yaml` (the curated, hand-maintained file):
   - `allowedTargets` was ALREADY correct (5 MVP targets only) — no change
     needed there.
   - `adapterTargets` was missing `vscode`/`codex` (only 8 of the 10
     targets declared, out of sync with `apps/cli/src/commands/init.ts`'s
     `ADAPTER_TARGETS`, which INC-20260726-adapter-registry-canonical
     extended to 10). Added both entries (`kind: ide`/`mvp: false` for
     `vscode`; `kind: cli`/`mvp: false` for `codex`), mirroring
     `DEFAULT_PROFILES`'s existing entries, so the curated file and the
     bundled default stay reconciled (per the file's own header comment:
     "Contenido idéntico al DEFAULT_PROFILES embebido").

3. Regression tests:
   - `Axiom/packages/install-profiles/tests/default-profiles.test.ts`:
     replaced the old assertion ("`resolveInstallProfile` succeeds for all
     10 targets, both profiles") with: (a) MVP targets still succeed for
     both profiles; (b) the 5 non-MVP targets now correctly `throw`
     (rejected — not in `allowedTargets`) for both profiles; (c) a new
     explicit IP-003-shaped assertion that every `profileBindings[*].
     allowedTargets` entry is in `MVP_TARGETS`; (d) `allowedTargets` is
     exactly the MVP set (as a set) for both profiles. `adapterTargets`
     still declares all 10 (unchanged assertion).
   - `Axiom/packages/installer/tests/installer.test.ts`: Scenario 6's
     "sigue funcionando con los 8 adapter targets" test asserted
     `installProfile` succeeds against the `DEFAULT_PROFILES` fallback for
     `cursor`/`github-copilot`/`litellm` (non-MVP) — this is now the
     opposite of correct behavior. Split into two tests: one asserting the
     5 MVP targets still succeed via the bundled fallback, one asserting
     the 5 non-MVP targets now correctly return `err({ kind:
     'composition-failed' })`.

## Known follow-up (NOT fixed here — out of this bug's scope)

Discovered while validating: `apps/cli/src/commands/workspace-adapters.ts`
(`generateWorkspaceAdapters`) also consumes `DEFAULT_PROFILES` directly
(line ~583), but purely in-memory — it never writes it to disk, so it has
no IP-003 exposure of its own. However, because it reuses the SAME
`DEFAULT_PROFILES` constant (now IP-003-restricted), calling
`generateWorkspaceAdapters` with a *primary* adapter that is
`vscode`/`codex`/`cursor`/`github-copilot`/`litellm` (and no project-level
`axiom.config/profiles.yaml` override) now causes `installProfile` to
return `composition-failed`, which `generateWorkspaceAdapters` degrades
into a "no files generated" warning for that repo — confirmed by running
`apps/cli/tests/workspace-adapters.test.ts`, where 3 pre-existing tests
("cursor/github-copilot/litellm...", "vscode escribe...", "codex escribe
un AGENTS.md real...") now fail. This is a genuine, real regression to
those specific dispatch paths, but fixing it requires a design decision
this bug's scope does not cover (e.g., should
`generateWorkspaceAdapters` synthesize its own wider `allowedTargets`
instead of reusing the IP-003-restricted `DEFAULT_PROFILES`, since it
never touches disk and IP-003 does not apply to it?). Recommend opening a
dedicated follow-up bug/increment for `apps/cli/src/commands/
workspace-adapters.ts` before this ships, since it silently drops adapter
file generation for these 5 non-MVP primary-adapter selections. NOT fixed
in this bug per "keep fixes minimal and targeted" and because the explicit
validation scope given for this bug was limited to `packages/
install-profiles` and `packages/doctor`.

## Validation

- `npm run build` (from `Axiom/`) — passes, no TypeScript errors.
- `npx vitest run packages/install-profiles packages/doctor` — 24 test
  files, 257 tests, all passing (27 tests in `default-profiles.test.ts`,
  including the new IP-003 regression assertions).
- `npx vitest run packages/install-profiles packages/doctor
  packages/installer` — 26 test files, 271 tests, all passing (confirms
  the updated `installer.test.ts` Scenario 6 split is consistent).
- `npx vitest run packages/doctor/tests/dogfooding.test.ts` — 8 tests,
  all passing (doctor self-check against this actual repo, confirming the
  curated `Axiom/axiom.config/profiles.yaml` edit — adding `vscode`/
  `codex` to `adapterTargets` — did not regress anything).
- Discovered-but-out-of-scope: `npx vitest run apps/cli/tests/
  workspace-adapters.test.ts` — 3 of 8 tests now fail (see "Known
  follow-up" above). Not part of this bug's declared validation scope
  (`packages/install-profiles packages/doctor`) and not fixed here.

## Result

Fixed. `DEFAULT_PROFILES`'s `allowedTargets` is now exactly the 5
`MVP_TARGETS` for both `product-owner` and `builder`, imported directly
from `@axiom/capability-model` so it cannot drift from IP-003 again. A
project scaffolded from the bundled default now passes IP-003 out of the
box. The curated `Axiom/axiom.config/profiles.yaml` was reconciled
(`adapterTargets` now also declares `vscode`/`codex`, matching
`DEFAULT_PROFILES` and `init.ts`'s `ADAPTER_TARGETS`; its `allowedTargets`
was already correct). All explicitly-scoped validation passes. A real,
separate regression was discovered in `apps/cli`'s
`generateWorkspaceAdapters` dispatch path (see "Known follow-up") and is
intentionally left open as a new candidate bug, not silently absorbed into
this fix.

## General spec integration

Stable knowledge worth consolidating: "`DEFAULT_PROFILES`'s
`profileBindings[*].allowedTargets` MUST always be derived from (or kept a
strict subset of) `@axiom/capability-model`'s `MVP_TARGETS` — never
hardcoded — because doctor's IP-003 enforces this invariant against
whatever `axiom.config/profiles.yaml` a project actually has on disk, and
the bundled default is what gets written there for any project without a
hand-curated override." This task's explicit instructions scope canonical
spec edits to this bug file only ("Do NOT edit specs/00..08") — this
repository's `Axiom.Spec/specs/` uses the numbered `00_..08_` files in
place of a single `general-spec.md`. The above is recorded here for a
maintainer to fold into `06_Integraciones_y_Capacidades.md` (or
equivalent) in a dedicated pass, scoped to spec-repo edits only.
