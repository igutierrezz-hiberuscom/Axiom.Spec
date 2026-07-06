# Bug: `axiom configure`/`axiom start` fail after `axiom init` because no `profiles.yaml` is ever created

Status: closed
Date: 2026-07-03

## Symptom

Running the documented onboarding flow (`axiom init` → `axiom join` →
`axiom configure` → `axiom sync` → `axiom start`) on a freshly initialized
project fails at `configure`/`start` with:

```
installProfile failed: No se pudo leer profiles.yaml en axiom.config/profiles.yaml: ENOENT: ...
```

(error kind `invalid-profiles-yaml`).

## Current behavior

`installProfile` (`packages/installer/src/installer.ts`) has no
`args.profilesData` in the normal CLI path (`configure.ts`, `start.ts`), so
it always calls `loadProfilesData(args.profilesYamlPath ??
'axiom.config/profiles.yaml')`. `axiom init` never creates
`axiom.config/profiles.yaml`, and no such file exists anywhere in the
bundled product (`@axiom/install-profiles` only exports constants/schemas/
composer — never a data catalog). The load always fails with ENOENT on any
project that only ran `init`.

## Expected behavior

Per the documented onboarding flow, `axiom configure` (and `axiom start`,
which derives an install profile the same way when
`install-profile.json` is missing) must succeed on a project that only ran
`axiom init` — **without** requiring the user to hand-author a
`profiles.yaml`. `profiles.yaml` content (functional profiles, overlays,
adapter targets, profile bindings, role aliases) is canonical **product**
data — identical across projects — so it should ship as a bundled default
inside `@axiom/install-profiles` and `installProfile` should fall back to
it. A project may still **override** the default by explicitly creating
its own `axiom.config/profiles.yaml`; that file, if present and readable,
takes precedence. `axiom init` itself must NOT scaffold/write a
`profiles.yaml` file (keeps the generated file count low) — it only needs
the same default for offline alias resolution (`--profile analista` /
`--profile arquitecto`) when no project file exists.

## Impact

Blocks the entire post-init onboarding flow (`configure`, `start`) for
every dogfood/test project that does not hand-create
`axiom.config/profiles.yaml`. No real projects exist yet, so no migration
is needed — this is a pure MVP-completeness gap.

## Reproduction steps

1. `axiom init` in an empty directory (any `--target`).
2. `axiom configure` (or `axiom start` without a pre-existing
   `install-profile.json`).
3. Observe `installProfile failed: No se pudo leer profiles.yaml en
   axiom.config/profiles.yaml: ENOENT ...`.

Reproduced directly by the pre-existing failing test
`apps/cli/tests/start.test.ts` → `Scenario 4: start sin
install-profile.json lo deriva de init.json` (this scenario's project
fixture does not create a real, non-empty `axiom.config/profiles.yaml` —
only `axiom.spec/config/profiles.yaml`, which is not the path
`installProfile` reads — so `start`'s internal `installProfile` call hits
the exact ENOENT above).

## Suspected cause (confirmed)

`profiles.yaml` is canonical product data, but nothing ships it:
- `axiom init` never writes `axiom.config/profiles.yaml`.
- No bundled default exists in `@axiom/install-profiles` (only
  `FUNCTIONAL_PROFILES`, `OPERATIONAL_OVERLAYS`, and other scalar
  constants — never an assembled `ProfilesYaml` object).
- `installProfile`'s `loadProfilesData` call has no fallback: any read
  failure (including plain ENOENT) becomes a hard `invalid-profiles-yaml`
  error.

## Acceptance criteria

- [x] `@axiom/install-profiles` exports a `DEFAULT_PROFILES: ProfilesYaml`
      constant that validates against `InstallProfilesYamlSchema`, covers
      both functional profiles (`product-owner`, `builder`), all 3
      operational overlays, and **all 8** CLI-supported adapter targets
      (`opencode`, `copilot-vscode`, `claude-code`, `antigravity`,
      `visual-studio-2026`, `cursor`, `github-copilot`, `litellm`) in both
      profiles' `allowedTargets`, plus `roleAliases: { analista:
      'product-owner', arquitecto: 'builder' }`.
- [x] `installProfile` falls back to `DEFAULT_PROFILES` when
      `args.profilesData` is absent and the resolved
      `axiom.config/profiles.yaml` path is not readable; it still prefers
      an explicit `args.profilesData` or a real, readable project-level
      `profiles.yaml` (override semantics preserved).
- [x] `axiom init`'s alias resolution (`resolveProfileToCanonical`) falls
      back to the same `DEFAULT_PROFILES` when no project/product
      `profiles.yaml` candidate is found, so `init --profile analista`
      and `init --profile arquitecto` work with zero project files.
- [x] `axiom init` does NOT write any `profiles.yaml` file.
- [x] `@axiom/doctor`'s GW-001 (`collectGatewayDigests`) does not
      crash/error when a project has no `axiom.config/profiles.yaml` (this
      is now the normal case) — confirmed already tolerant
      (`readFileContent` → `null` → `"...|0|missing"` marker, no throw).
- [x] `apps/cli/tests/start.test.ts` Scenario 4 passes.
- [x] Zero new test failures; the pre-existing unrelated failing set
      (10 tests, listed below) is unaffected.
- [x] Build (`npm run build`) is clean.

## Fix notes

Implemented exactly the shape above:

1. `Axiom/packages/install-profiles/src/default-profiles.ts` (new file) —
   `DEFAULT_PROFILES: ProfilesYaml`, built by reconciling the validated
   fixture in `packages/install-profiles/tests/composer.test.ts` with
   `constants.ts` (`FUNCTIONAL_PROFILES`, `OPERATIONAL_OVERLAYS`,
   `CODE_DOMAIN_CAPABILITIES`, `COMPLIANCE_RISK_BY_OVERLAY`,
   `DISCOVERY_PROFILE_BY_OVERLAY`) and widened to all 8 adapter targets
   (both profiles' `allowedTargets`) plus `roleAliases`. Exported from
   `src/index.ts`.
2. `Axiom/packages/installer/src/installer.ts` — when `args.profilesData`
   is absent, `installProfile` now tries `loadProfilesData` at the
   resolved path (joined against `args.projectRoot` when relative) and
   falls back to `DEFAULT_PROFILES` on any read failure, instead of
   propagating `invalid-profiles-yaml`.
3. `Axiom/apps/cli/src/commands/init.ts` — `resolveProfileToCanonical`'s
   caller now falls back to `DEFAULT_PROFILES` when no candidate
   `profiles.yaml` path loads, so alias resolution never fails purely due
   to a missing file.
4. No doctor code change was needed — `collectGatewayDigests` already
   treats a missing file as `"<path>|0|missing"` (verified by direct
   reading; no doctor test asserts profiles.yaml existence for GW-001).

## Validation

`npm run build` from `Axiom/`: clean, zero errors.

`npx vitest run` (full suite) tail:

```
Test Files  10 failed | 153 passed (163)
     Tests  10 failed | 1662 passed (1672)
```

`start.test.ts` Scenario 4 now passes. The remaining 10 failing tests are
the pre-existing, unrelated ones (fixture-absence / date-dependent, all
present in the baseline run before this fix):
- `packages/agents/tests/catalog.test.ts` — real-repo catalog load
- `packages/doctor/tests/checks.test.ts` — real-repo project resolution
- `packages/model-routing/tests/assignments.test.ts` — real policy file
- `packages/model-routing/tests/loader.test.ts` — real policy file
- `packages/model-routing/tests/opencode-projection.test.ts` — slot count
- `packages/model-routing/tests/resolver.test.ts` — real policy file
- `packages/project-resolution/tests/resolver.test.ts` — v1 scope resolution
- `packages/skills/tests/catalog.test.ts` — real-repo skills catalog
- `packages/telemetry/tests/audit-trail-sink.test.ts` — retention window (date-dependent)
- `packages/toolchain/tests/repair-add-gitignore.test.ts` — gitignore repair fixture

Baseline (before fix) was 11 failed / 1661 passed — the exact same 10
above, plus `start.test.ts` Scenario 4. Net effect: -1 failure, 0 new
failures.

## Result

Fix implemented and validated. `configure`/`start` now succeed on a
project that only ran `axiom init`, using the bundled `DEFAULT_PROFILES`
when no project-level override exists. A project can still override by
creating its own `axiom.config/profiles.yaml`.

## General spec integration

Integrated into `Axiom.Spec/specs/03_Modelo_Operativo_y_Datos.md`: added
a note documenting that `profiles.yaml` ships as a canonical bundled
default (`@axiom/install-profiles`'s `DEFAULT_PROFILES`) used whenever a
project has no `axiom.config/profiles.yaml`, and that a project may
override it by creating that file explicitly. `axiom init` does not
scaffold it.
