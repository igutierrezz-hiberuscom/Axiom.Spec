# Increment: AutoSkills lock/catalog hygiene (provenance + date + policy gate)

Status: closed
Date: 2026-07-24

## Goal

Improve AutoSkills supply-chain hygiene so community/autoskills-installed
skills are identifiable and governed: add a distinct provenance marker, an
install date/timestamp, and an allow/deny/license policy gate — without
breaking existing behavior.

This is **INC-S1** of Cluster S in the Axiom evolution plan
(`giggly-imagining-moore.md`, "Cluster S → INC-S1"), Wave 2 (parallel with
Cluster T; Wave 1 + Cluster T already done). User's intent: AutoSkills stays
for install-time, per-code-repo skill suggestions, but the installed skills
must be identifiable as community/AutoSkills-sourced and governed by policy.
Explicitly user-approved; not down-scoped.

## Context

Before this increment, `apps/cli/src/commands/workspace-autoskills.ts`
(`installSuggestedSkills`) detected a code repo's stack and wrote suggested
skill entries into `axiom.config/skills-catalog.yaml` (the `CatalogEntry`
shape owned by `@axiom/skills`'s `catalog.ts`: `id/name/version/source/
status/securityCheckStatus/bundleHash`) with no marker distinguishing them
from the seed entries `workspace-skills.ts`'s `scaffoldCodeRepoSkills`
writes, no install timestamp, and no policy check — any suggested id that
existed in the curated `STACK_SKILL_MAP` was always installed.

Two important pre-existing facts shaped the design:

1. **`source` is already taken.** `CatalogEntry.source` holds the path to
   the bundled `.md` file, relative to the project root — it is NOT, and
   was never, a provenance tag. Any new field had to be additive and
   distinct.
2. **Two unrelated "skill lock" surfaces exist in this codebase**, and only
   one of them is what AutoSkills actually writes:
   - `axiom.config/skills-catalog.yaml` (`CatalogEntry`, `@axiom/skills`) —
     the catalog `workspace-autoskills.ts`/`workspace-skills.ts` write to
     and `applySkillSet`/`materializeSkillSet` read from. **This is the
     surface this increment enriches.**
   - `axiom.skills.lock` (repo root, `LockfileEntry` shape: `skillId/
     source/version/bundleHash/reviewStatus/securityCheckStatus/
     installedTo/installedAt/portableEntry`) — a **different**, doctor-
     governed artifact (`packages/doctor/src/governance-checks.ts`'s
     GC-001…GC-013) used for Axiom's own dogfooding scaffold. Nothing in
     the AutoSkills flow reads or writes this file; it already carries its
     own (pre-existing, unrelated) `installedAt` field. **Left untouched**
     — see Assumptions.

`installedAt` as a field name was independently confirmed as the
established convention for "when a skill entry was installed" across
*two* other pre-existing surfaces in this codebase: the opencode adapter's
`.opencode/skills-lock.yaml` (read by `@axiom/skills`'s `loader.ts`) and
the governance lockfile above — reusing the same name for the catalog's
new field keeps terminology consistent repo-wide.

## Scope

- `packages/skills/src/catalog.ts`: new `SkillProvenance` union type
  (`'autoskills' | 'axiom-native' | 'project-native' | 'user-local'`), new
  `DEFAULT_SKILL_PROVENANCE` constant (`'axiom-native'`), two new *optional*
  `CatalogEntry` fields (`provenance?`, `installedAt?`), and parsing/
  validation logic in `loadSkillsCatalog` (default `provenance` when
  absent; validate the closed union when present; validate `installedAt`
  is a non-empty string when present; never invent `installedAt`).
- `packages/skills/src/index.ts`: barrel export of `SkillProvenance` +
  `DEFAULT_SKILL_PROVENANCE`.
- `apps/cli/src/commands/workspace-autoskills.ts`:
  - `license` field added to the 5 curated `STACK_SKILL_METAS` (all
    `'axiom-curated'` — 100% Axiom-authored/bundled content).
  - New policy gate: `AutoskillsPolicyConfig`, `DEFAULT_AUTOSKILLS_POLICY`
    (allow-all), `AutoskillsPolicyCandidate`, `AutoskillsPolicyDecision`,
    `loadAutoskillsPolicy` (reads an optional
    `axiom.config/autoskills-policy.yaml` from the code repo; degrades to
    allow-all on any absence/malformation), `evaluateAutoskillsPolicy`
    (pure: denylist → license → allowlist).
  - `InstallSuggestedSkillsArgs.now?: string` — injectable clock, threaded
    once (`args.now ?? new Date().toISOString()`), never invented deeper
    in the call chain.
  - `InstallSuggestedSkillsResult.policySkipped` — new field: skills the
    policy gate denied, each with a clear `reason` (also mirrored into
    `warnings` for CLI/wizard visibility with zero changes needed in
    `skills.ts`'s output formatting).
  - `installSuggestedSkills` now evaluates the policy gate per candidate
    (after the existing unknown-id/no-clobber checks, before writing
    anything) and writes `provenance: autoskills` + `installedAt: "<now>"`
    on every entry it installs.
- Tests: `packages/skills/tests/catalog.test.ts` (Scenario 7, 10 new
  tests), `apps/cli/tests/workspace-autoskills.test.ts` (20 new unit
  tests: `loadAutoskillsPolicy`, `evaluateAutoskillsPolicy`,
  provenance/installedAt, policy gate), `apps/cli/tests/skills.test.ts`
  (Scenario 8, 5 new e2e tests through the real `runSkillsSuggest --apply`
  CLI surface).

## Non-goals

- `axiom.skills.lock` (repo-root governance lockfile) — untouched; see
  Context/Assumptions for why it is a different, unrelated surface.
- RTK/concision skills (INC-T2/INC-T3, closed), `cmm`/providers (INC-T1,
  closed), worktrees (Cluster W), the unified `<project>.axiom` MCP
  (Cluster A) — untouched.
- Migrating Axiom's own repos (`Axiom.SDD`/`Axiom.Spec`) to any new model
  — out of scope (deferred to the user per the plan).
- A rich policy engine (rule DSL, per-skill overrides beyond id/license,
  remote policy fetch). The gate is deliberately a flat denylist +
  allowlist + license-allowlist, config-driven, allow-all by default.
- Any change to *when* AutoSkills runs (still per-code-repo, at install
  time only — `workspace setup`'s wizard phase and `axiom skills suggest
  --apply`; no pull/session/worktree hook was added).

## Acceptance criteria

- [x] Autoskills-installed catalog entries carry a distinct provenance
      marker (`provenance: 'autoskills'`), without altering the existing
      meaning of `source`.
- [x] The new `provenance` field is backward compatible: entries written
      before this field existed (no `provenance` in their YAML) load fine
      and default to `'axiom-native'`.
- [x] An install date/timestamp (`installedAt`, ISO-8601) is recorded on
      autoskills-installed entries, threaded via an injectable clock
      (`InstallSuggestedSkillsArgs.now`) — never invented deeper in the
      call chain; entries that don't track it (e.g. pre-existing seeds)
      leave it `undefined`, never defaulted/invented.
- [x] An allow/deny/license policy gate is applied during
      `installSuggestedSkills`; a denied or disallowed-license skill is
      skipped with a clear, human-readable reason
      (`policySkipped` + `warnings`) and is never installed.
- [x] The policy source is simple and config-driven (a new, 100% optional
      `axiom.config/autoskills-policy.yaml` in the code repo; absent or
      malformed ⇒ sensible allow-all default, never over-built).
- [x] AutoSkills still runs per-code-repo at install time only (unchanged);
      degrades gracefully (never blocks installation) if the policy file
      or AutoSkills detection is unavailable/malformed.
- [x] `npm run build` (`tsc -b`) passes.
- [x] Targeted `vitest run` for `packages/skills`, `packages/doctor`,
      `packages/mcp-tools`, and the full `apps/cli` suite passes with 0
      new failures (0 pre-existing failures observed either).
- [x] Guard-test lockstep reviewed: `packages/doctor/tests/skills.test.ts`
      (TC-010) and `packages/doctor/tests/governance.test.ts` (GC-001…013)
      both stay green unmodified — neither check depends on the new
      optional fields, and Axiom's own `axiom.config/skills-catalog.yaml`
      (the 20-entry catalog they assert against) was intentionally left
      untouched by this increment.
- [x] New unit tests: autoskills-installed entry gets `provenance:
      autoskills` + `installedAt`; policy denylist skips a skill with a
      clear reason; license policy is respected; backward-compat for
      entries missing `provenance` verified.
- [x] e2e test added: onboarding a sandbox code repo (via
      `runSkillsSuggest --apply`, the real per-code-repo-at-install-time
      surface) suggests + installs only policy-allowed skills; a denied
      skill is skipped; installed entries carry `provenance: autoskills` +
      `installedAt`.

## Open questions

None blocking — ambiguities were resolved using existing-repo precedent
(recorded below in Assumptions) rather than raised as blockers.

## Assumptions

Each is a deliberate, narrowest-change resolution of an ambiguity the
brief left open, using existing-precedent-first:

1. **Which "lock" this increment enriches.** The brief's file list named
   both `workspace-autoskills.ts`/`packages/skills` AND `axiom.skills.lock`.
   Ground-truth inspection showed these are two unrelated mechanisms (see
   Context) and nothing in the AutoSkills flow ever reads/writes
   `axiom.skills.lock`. Enriching the catalog (`axiom.config/
   skills-catalog.yaml`, via `@axiom/skills`) is the change that actually
   affects AutoSkills-installed skills; `axiom.skills.lock` was left
   untouched as out of scope (it is Axiom's own dogfooding governance
   artifact, and the plan explicitly defers migrating Axiom's own repos).
2. **`provenance` optional at the type level, always resolved at runtime.**
   Making it a required `CatalogEntry` field would have broken every
   pre-existing object-literal construction site across the codebase
   (`packages/skills/tests/materialize.test.ts`'s fixtures, in particular)
   that predates this field. Keeping it `provenance?: SkillProvenance` at
   the type level preserves those call sites; `loadSkillsCatalog` still
   always resolves a concrete value (defaulting to `axiom-native`) for any
   catalog it actually loads from disk, so real consumers never see
   `undefined`.
3. **Policy file location: a new, dedicated, code-repo-local file**
   (`axiom.config/autoskills-policy.yaml`), not an extension of the
   existing shared `axiom.config/policy-as-code.yaml`. The latter already
   has its own doctor-governed contract (`PC-002`, presence-only check)
   and unrelated concerns (project isolation, MCP catalog, mutation
   guard); coupling a skill-license policy to it would widen the blast
   radius for no benefit. A dedicated, 100%-optional file keeps the change
   surgical and trivially backward compatible (absent ⇒ allow-all).
4. **License field: `'axiom-curated'` for all 5 curated stack skills.**
   These are 100% Axiom-authored/bundled markdown (never fetched from a
   third-party registry — see the file's own header). Giving them a real,
   concrete `license` value (rather than hardcoding a magic string only
   inside the policy evaluator) makes the license gate meaningfully
   testable without inventing external dependencies.
5. **Clock: `InstallSuggestedSkillsArgs.now?: string`, defaulted once.**
   Mirrors the exact existing pattern already used by `@axiom/workflow`'s
   `artifact-store.ts` (`scaffoldArtifact`'s `options.now ?? new
   Date().toISOString()`). No production call site
   (`runSkillsSuggest`/`runAutoskillsWizardPhase`) needed to change; both
   already omit it and get real time by default. Tests inject a fixed
   value for deterministic assertions.
6. **`policySkipped` as a dedicated, structured result field**, in
   addition to (not instead of) a human-readable line in `warnings`. This
   gives both programmatic callers (tests, future tooling) and end users
   (CLI/wizard output, which already forwards `warnings` verbatim) a clear
   answer to "what got skipped and why" without any change needed in
   `apps/cli/src/commands/skills.ts`'s existing output formatting.

## Implementation notes

All files changed live in `Axiom/` (the product monorepo) unless noted.

- Changed: `packages/skills/src/catalog.ts` (new `SkillProvenance` type +
  `DEFAULT_SKILL_PROVENANCE` const + 2 new optional `CatalogEntry` fields +
  parsing/validation), `packages/skills/src/index.ts` (barrel export),
  `apps/cli/src/commands/workspace-autoskills.ts` (license field, policy
  gate module, `now` threading, `policySkipped`, wiring into
  `installSuggestedSkills`).
- Test files changed: `packages/skills/tests/catalog.test.ts` (+10 tests,
  Scenario 7, including a 4-case `it.each`), `apps/cli/tests/
  workspace-autoskills.test.ts` (+20 tests), `apps/cli/tests/skills.test.ts`
  (+5 tests, Scenario 8).
- Explicitly NOT changed (see Scope/Non-goals/Assumptions for why):
  `axiom.skills.lock`, `axiom.config/skills-catalog.yaml` (Axiom's own,
  20-entry catalog — no AutoSkills-installed entries exist there),
  `axiom.config/policy-as-code.yaml`, `packages/doctor/*` (no doctor
  check needed updating — verified both TC-010 and GC-001…013 stay green
  unmodified), `@axiom/cli-commands`, `@axiom/tui`, any git hook.
- No new production dependency: `js-yaml` was already a dependency of
  `@axiom/cli` (used by many sibling command files) and of `@axiom/skills`.

## Validation

Ran from `C:\repos\Axiom Workspace\Axiom`:

- `npm run build` (`tsc -b`) — PASSED, clean, exit 0, no output.
- Targeted `npx vitest run packages/skills` — **6 files, 85 tests
  passed** (up from 75 pre-existing; the 10 new Scenario 7 tests, one of
  which is a 4-case `it.each`).
- Targeted `npx vitest run packages/doctor` — **18 files, 182 tests
  passed**, unchanged count — confirms TC-010 and GC-001…013 both stay
  green without modification.
- Targeted `npx vitest run packages/mcp-tools` — **12 files, 89 tests
  passed** (consumer of `CatalogEntry`'s type; unaffected).
- Targeted `npx vitest run` on `apps/cli/tests/{workspace-autoskills,
  skills,tui,workspace-skills}.test.ts` — **4 files, 104 tests passed**
  (44 + 13 + 30 + 17).
- Full `npx vitest run apps/cli` — **116 files, 1099 tests passed. Zero
  failures.**

No pre-existing failures were observed to classify — the suite was fully
green before and after in every scope run.

## Result

Implemented. AutoSkills-installed catalog entries are now distinguishable
from Axiom-native/seed entries via a new, additive `provenance` field
(`'autoskills'`), carry an `installedAt` ISO-8601 timestamp threaded from
an injectable clock (never invented), and pass through a config-driven
allow/deny/license policy gate before being written — a denied skill is
skipped with a clear reason, surfaced both programmatically
(`policySkipped`) and to the CLI/wizard user (`warnings`). The gate is
100% optional and config-driven (`axiom.config/autoskills-policy.yaml`);
its absence, or any malformation, degrades to the exact allow-all
behavior AutoSkills had before this increment — never blocking
installation. Every pre-existing entry (seeded by `scaffoldCodeRepoSkills`
or authored before this increment) continues to load unchanged, defaulting
to `provenance: 'axiom-native'` with no `installedAt`. Full build + all
targeted suites (including the two doctor guard-check families,
`packages/mcp-tools`, and the full `apps/cli` suite) are green with zero
regressions.

## General spec integration

Integrado en la spec canónica al cierre del batch INC-20260724-* (integración
única de toda la tanda). Ficheros que recibieron el conocimiento de este
incremento:

- `06_Integraciones_y_Capacidades.md` — `provenance`/`installedAt` en
  `CatalogEntry` + gate de policy allow/deny/licencia
  (`autoskills-policy.yaml`, allow-all por defecto).
- `07_Gobierno_y_Seguridad.md` — higiene de supply-chain de AutoSkills
  (identificable, fechada, gobernada por policy).
- `08_Glosario.md` — `provenance`/`installedAt` (skill lock).
- `01_Requisitos_Funcionales.md` — RF-AXM-046.
