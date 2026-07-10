# Increment: Wizard/configure provider selection (AB6)

Status: closed
Date: 2026-07-08

## Goal

Make provider selection EXPLICIT in both the TUI workspace wizard and
`axiom configure`: let the user choose WHICH LOCAL providers
(`codegraph`, `serena`, `graphify`, `engram`) are enabled for a project,
persist that choice, and leave the chosen providers WIRED + READY so
`invokeCapabilityLive`/MCP handlers actually route to them. This kills
the "declared-not-wired / false breadth" debt AB4 explicitly deferred:
`enabledCodeIntelProviders` used to be a plain, caller-supplied opt-in
list with no real selection UX or persisted, single source of truth.
LOCAL-only; only the chosen providers are enabled — a fresh project's
persisted state never implies more breadth than the user actually asked
for.

## Context

- `@axiom/providers` (AB3): `ProviderRegistry`, `invokeCapabilityLive`,
  local stdio client, `LOCAL_ONLY`/`isLocalTarget`.
- AB4 (`INC-20260708-code-intel-providers-wired`):
  `registerCodeIntelProviders(registry, opts?)` wires real
  `ProviderClient`s for `codegraph`/`serena`/`graphify`.
  `WorkspaceSetupSpec.enabledCodeIntelProviders` is a plain, caller-
  supplied opt-in list (no selection UX, no persistence contract) threaded
  into a best-effort `initializeCodeIntelIndexes` hook and into the
  canonical `AGENTS.md`'s "Code intelligence tools" guidance section.
  That increment's own closing note explicitly named this increment
  (AB6) as the one that should "plug a real, persisted/resolved value
  into" that seam.
- AB5 (`INC-20260708-memory-real-local-backend`): `@axiom/memory`'s
  `resolveMemoryBackend(projectRoot, scope, opts?)` probes a LOCAL
  `engram mcp` process and falls back to the JSON backend — engram is
  NOT a `ProviderClient` (it has its own `MemoryBackend` interface and is
  resolved directly by `memory.decisionRecall`/`memory.contextRecall`'s
  MCP handlers, not through `ProviderRegistry`/`invokeCapabilityLive`).
- `axiom.config/providers.yaml` (`@axiom/capability-model`) is a
  **schema-locked, closed registry of exactly 7 canonical provider ids**
  (`ProviderRegistrySchema`: `.length(7)`; `CC-001`/`CC-003` doctor
  checks fail the project if the count/id-set drifts from
  `CANONICAL_PROVIDER_IDS`, per ADR-0021). It describes what providers
  EXIST in the model (their capability mapping, fallback chain,
  `mvpDefault`), not what one project has chosen to enable. **This means
  the brief's literal wording ("trim `providers.yaml` to reflect only the
  enabled providers") is not compatible with the schema/doctor
  invariants as they exist today** — doing so would break `CC-001`
  (`Registry must contain exactly 7 providers`) for every project that
  enables fewer than all optional providers, which is the common case.
  Resolved (see Assumptions): persist the ENABLED-providers selection as
  project-scoped STATE (`.axiom-state/<projectId>/workspace.json`'s new
  `providers` array — same anchor `adapters`/`profile`/`overlay` already
  use), not by mutating the canonical, doctor-validated `providers.yaml`.
  `providers.yaml` remains the closed 7-provider MODEL; the new state is
  the project's SELECTION over that model. This removes the false-breadth
  concern the same way trimming would have (a fresh project's
  `workspace.json` declares only what it enabled) without breaking a
  schema invariant three prior increments (AB3/AB4/AB5) and `doctor`
  already depend on.
- The TUI workspace wizard (`apps/cli/src/commands/tui.ts`'s
  `buildWorkspaceSetupWizardSteps`/`buildWorkspaceSetupSpecFromWizard`)
  already has one `multi-select` step (`adapters`) whose answer
  serializes as a comma-joined string in `WizardResult.values['adapters']`
  — the exact same parsing idiom (`split(',')`) is reused for the new
  `providers` step.
- `axiom configure` (`apps/cli/src/commands/configure.ts`) materializes
  the install profile + adapter files from `init.json`/`profiles.yaml`;
  it had no provider-selection surface at all.
- `packages/capability-model/src/constants.ts`'s `CANONICAL_PROVIDER_IDS`
  is the 7-id closed set (`filesystem`, `axiom-gateway`, `serena`,
  `codegraph`, `graphify`, `engram`, `generated-snapshots`) — the
  SELECTABLE subset for this increment's UX is narrower: `filesystem` is
  always-on (never offered, it is the baseline/no-op provider with no
  external tool to enable/disable), `axiom-gateway`/`generated-snapshots`
  are non-local/broker/degraded-mode markers with no `ProviderClient`
  today and are out of scope for a LOCAL-only selection UX.

## Scope

- `packages/providers/src/selectable.ts` (new) — `SELECTABLE_PROVIDER_IDS`
  (`readonly ['codegraph', 'serena', 'graphify', 'engram']`, the single
  source of truth for "which providers this selection UX offers"),
  `isSelectableProviderId`.
- `packages/providers/src/project-registry.ts` (new) —
  `buildProjectProviderRegistry(projectRoot, opts?)`: reads a project's
  enabled-providers list (via `readEnabledProviders`, checking
  `workspace.json` across every `.axiom-state/*/` project dir, else
  `opts.enabledProviders` override for callers that already resolved it)
  and returns a `ProviderRegistry` with exactly the corresponding LOCAL
  code-intel clients registered (`registerCodeIntelProviders`, filtered
  to the enabled subset) plus the always-on `filesystem` client. `engram`
  is intentionally NOT registered here as a `ProviderClient` (it has no
  `ProviderClient` implementation — AB5 built it as a `MemoryBackend`,
  resolved through `resolveMemoryBackend`, not `ProviderRegistry`); the
  function instead reports `engramEnabled: boolean` in its result so a
  caller (doctor, docs) can reason about it uniformly alongside the
  code-intel providers without inventing a fake `ProviderClient` shim for
  a backend that already has its own, better-fitting resolution path.
- `packages/providers/src/index.ts` — barrel exports for the above.
- `packages/providers/package.json` — no new dependency (reuses
  `js-yaml`... actually not needed: `buildProjectProviderRegistry` reads
  `workspace.json`, which is plain JSON via `fs`, no YAML parsing
  required).
- `apps/cli/src/commands/tui.ts` — new `providers` `multi-select` step in
  `buildWorkspaceSetupWizardSteps` (options = `SELECTABLE_PROVIDER_IDS`;
  default preselect: NONE); `buildWorkspaceSetupSpecFromWizard` parses it
  into `WorkspaceSetupSpec.enabledProviders`.
- `apps/cli/src/commands/workspace-setup.ts` — new
  `WorkspaceSetupSpec.enabledProviders?: readonly string[]` field
  (supersedes `enabledCodeIntelProviders` as the primary selection input;
  see Assumptions for the back-compat/precedence rule).
  `workspace.json`'s persisted record gains a `providers: string[]`
  field. The AB4 code-intel init hook (`initializeCodeIntelIndexes`) is
  now driven by the code-intel subset of `enabledProviders` (falling back
  to `enabledCodeIntelProviders` when `enabledProviders` is omitted, for
  any direct caller that has not migrated yet); a new best-effort engram
  probe step covers `memory` initialization the same way (probes
  `resolveMemoryBackend`, best-effort, never fails setup).
- `apps/cli/src/commands/configure.ts` — new `--providers <csv>` CLI flag
  and a matching `ConfigureArgs.providers?: readonly string[]` field;
  when present, persists the selection into BOTH
  `.axiom-state/<projectId>/workspace.json` (merges into the existing
  record, creating one if absent) — configure's existing
  installProfile/adapter-generation behavior is otherwise untouched.
- `packages/document-bootstrap/src/canonical-agents-md.ts` — new
  `CanonicalAgentsMdIdentity.enabledMemoryProvider?: boolean` (engram
  guidance line) rendered alongside the existing code-intel block under
  the same "Code intelligence tools" heading (renamed in spirit, not
  literally, to keep the existing AB4 tests' string assertions intact —
  see Implementation notes for the exact heading decision).
- `packages/doctor/src/checks.ts` + `packages/doctor/src/index.ts` +
  `packages/doctor/package.json` — new `runProviderSelectionChecks`
  (`PS-001`): reads the active project's enabled-providers selection
  (`workspace.json`) and reports, per enabled provider, whether a real
  `ProviderClient`/backend is registered/probable (uses
  `buildProjectProviderRegistry` for code-intel ids; a lightweight
  `engram` probe reusing `@axiom/memory`'s `resolveMemoryBackend` for the
  memory id) — `pass` when enabled-and-registered, `warn` when
  enabled-but-not-installed (never `fail`: a missing local dev tool is
  not a project defect), `pass` with an informational note when nothing
  is enabled.
- Tests: `packages/providers/tests/selectable.test.ts`,
  `packages/providers/tests/project-registry.test.ts`,
  new scenarios in `apps/cli/tests/tui.test.ts` (`providers` step +
  wizard end-to-end persistence), `apps/cli/tests/workspace-setup.test.ts`
  (new Scenario (k): `enabledProviders` -> `workspace.json`), new
  scenarios in `apps/cli/tests/configure.test.ts` (`--providers` flag
  persistence), new scenarios in
  `packages/document-bootstrap/tests/canonical-agents-md.test.ts`
  (engram guidance line), new `packages/doctor/tests/provider-selection.test.ts`.

## Non-goals

- Does NOT touch `axiom.config/providers.yaml`'s schema, its 7-provider
  closed-set invariant, or `CC-001`/`CC-002`/`CC-003` doctor checks —
  the brief's literal "trim providers.yaml" instruction is superseded by
  the project-scoped-state approach documented in Context/Assumptions;
  this is a deliberate, explicitly-noted deviation, not an oversight.
- No full per-command wiring of `buildProjectProviderRegistry` into every
  CLI command/skill/agent invocation path — `invokeCapabilityLive`
  callers are not yet threaded through the codebase (confirmed: no
  existing caller of `invokeCapabilityLive` exists outside tests as of
  this increment; AB3/AB4/AB5 built the seam and the clients, but no
  command yet routes real tool calls through it). This increment exposes
  the single entry point (`buildProjectProviderRegistry`) + wires it into
  ONE real consumer (the new doctor check) per the brief's "at minimum
  expose it + one real consumer" floor; full command/skill/agent routing
  is noted as follow-up.
- No new `ProviderClient` for `engram` — engram's `MemoryBackend`
  interface (AB5) is a deliberately different, better-fitting shape for
  a project-scoped-memory backend; forcing it through `ProviderClient`
  would mean building a synthetic adapter with no real consumer today.
- No changes to `@axiom/tool-routing`'s resolution logic, to
  `providers.yaml`'s fallback chains, or to `capabilities.yaml`.
- No new npm dependencies.
- No integration into `Axiom.Spec/specs/00-08` (reserved for the
  orchestrator's final consolidation pass) and no git commits.

## Acceptance criteria

- [x] Wizard: a `providers` `multi-select` step exists in
      `buildWorkspaceSetupWizardSteps`, options =
      `codegraph`/`serena`/`graphify`/`engram`, default preselect: none,
      placed after `adapters`. Selecting e.g. `codegraph,engram` ->
      `WorkspaceSetupSpec.enabledProviders === ['codegraph', 'engram']`.
- [x] `runWorkspaceSetup` persists `enabledProviders` into
      `.axiom-state/<projectId>/workspace.json`'s new `providers` field;
      selecting none persists `providers: []` (only `filesystem` is
      implicitly always-on, never listed since it is not offered/not
      opt-in).
- [x] The AB4 code-intel init hook (`initializeCodeIntelIndexes`) runs
      driven by `enabledProviders`'s code-intel subset (`codegraph`/
      `serena`/`graphify`); a new best-effort engram init/probe step
      covers `engram` the same way, gated on `enabledProviders` including
      `'engram'`, never failing setup.
- [x] `buildProjectProviderRegistry(projectRoot, opts?)` exists in
      `@axiom/providers`, registers exactly the enabled code-intel
      clients (reading the selection from `workspace.json` when
      `opts.enabledProviders` is not explicitly supplied), always
      registers `filesystem`, and reports which providers were
      requested-but-absent-from-the-registry (unknown/unregistered ids
      degrade at `invoke()` time via the existing `not-installed`
      mechanism — never thrown).
- [x] `axiom configure --providers <csv>` persists the selection into
      `workspace.json` (creating the file if absent) without altering
      configure's existing installProfile/adapter-generation behavior.
- [x] A doctor check (`PS-001`) reports, for the active project, each
      enabled provider vs. whether it is registered
      (code-intel/`buildProjectProviderRegistry`) or probably-installed
      (engram, via a lightweight `resolveMemoryBackend` probe) — `warn`
      (not `fail`) when enabled-but-not-installed.
- [x] The canonical `AGENTS.md`'s guidance section reflects the ENABLED
      providers including engram (extends AB4's mechanism; does not
      regress the existing code-intel-only rendering when no memory
      provider is enabled).
- [x] Hermetic tests cover: wizard parsing + persistence + trimmed
      selection (none selected -> only filesystem), registry
      registration (enabled vs. absent), `configure --providers`
      persistence, and the doctor check's enabled-vs-installed report.
      None depend on real `codegraph`/`serena`/`graphify`/`engram`
      binaries being installed.
- [x] `npm run build` (`tsc -b`) clean.
- [x] `npx vitest run apps/cli packages/providers packages/tui
      packages/doctor` passes.
- [x] `npx vitest run --no-file-parallelism` (full suite) confirms 0
      regressions vs. the AB5 baseline (`1936/1936`).

## Open questions

None blocking. Ambiguities resolved under Assumptions.

## Assumptions

- **`providers.yaml` is NOT trimmed; project-scoped state
  (`workspace.json`) is the persistence target instead.** The brief
  explicitly asked to "trim/write `axiom.config/providers.yaml` to
  reflect ONLY the enabled providers" — reading the actual schema
  (`ProviderRegistrySchema` in `packages/capability-model/src/schemas.ts`)
  shows `providers: z.array(...).length(7, { message: 'Registry must
  contain exactly 7 providers' })`, enforced again at `doctor` level by
  `CC-001`/`CC-003` (`packages/doctor/src/checks.ts`,
  `packages/doctor/tests/capability-model.test.ts`). Any project that
  enabled fewer than all 4 selectable providers (the overwhelmingly
  common case — nobody installs codegraph+serena+graphify+engram by
  default) would fail `CC-001` on its very next `axiom doctor` run if
  `providers.yaml` were trimmed. This is a hard schema/doctor
  contract three prior CLOSED increments (AB3/AB4/AB5) already depend
  on; breaking it here (even if "the brief said so") would be a
  regression, not a feature. The project-scoped-state approach achieves
  the SAME practical outcome the brief wanted (no false breadth: a
  fresh project's persisted state names only what it enabled) without
  touching the closed-set model file. This is the single most
  significant resolved ambiguity in this increment and is intentionally
  over-documented here for the final consolidation pass.
- **`enabledProviders` supersedes `enabledCodeIntelProviders` with a
  precedence rule, not a hard removal**: `WorkspaceSetupSpec` keeps BOTH
  fields for one increment (removing a field a currently-closed prior
  increment introduced, with no deprecation window, would be needlessly
  disruptive to any external caller). `runWorkspaceSetup` computes the
  effective code-intel list as `spec.enabledProviders !== undefined ?
  spec.enabledProviders.filter(isCodeIntelId) : (spec.
  enabledCodeIntelProviders ?? [])` — i.e. once a caller passes
  `enabledProviders` (even `[]`), it takes over entirely; the OLD field
  is only consulted when the NEW one is entirely absent. The TUI wizard
  (the only in-repo caller of either field) is migrated to always pass
  `enabledProviders` (never `enabledCodeIntelProviders`), so this
  precedence only matters for hypothetical external/future callers.
- **Selectable set = 4 ids, not the full 7-id `CANONICAL_PROVIDER_IDS`**:
  `filesystem` is the always-on, no-external-tool baseline (never
  offered — enabling/disabling it has no real effect since it needs no
  local binary and every other capability already falls back to it).
  `axiom-gateway` and `generated-snapshots` are non-local
  broker/degraded-mode markers with no `ProviderClient` and no
  selection-relevant "installed locally?" question to ask (`axiom-gateway`
  is explicitly forbidden under the `local-only` overlay this whole
  batch operates under; `generated-snapshots` is a fallback marker, not
  something a user "installs"). `SELECTABLE_PROVIDER_IDS` is therefore
  exactly `['codegraph', 'serena', 'graphify', 'engram']` — the 4 ids
  that map 1:1 to a real, optionally-installed LOCAL tool.
- **Default preselect: none.** Selected over "a sensible minimal" default
  (e.g. preselecting `serena`, which is `mvpDefault: true` in
  `providers.yaml`) because (a) `serena`'s `mvpDefault: true` already
  means `code.semanticNavigation` resolves to it in the CAPABILITY MODEL
  regardless of this UX's selection (the model's own fallback chain,
  unaffected by this increment, already prefers it); (b) the whole point
  of this increment is an EXPLICIT, opt-in choice with zero implied
  breadth — a friendlier "recommended: serena" hint belongs in the
  step's `subtitle` copy (added), not in a silently-pre-checked default
  a user might not notice and ship un-reviewed.
- **`buildProjectProviderRegistry`'s selection source**: reads
  `workspace.json` from EVERY `.axiom-state/<projectId>/` directory
  under `projectRoot` (there is normally exactly one; `fs.readdirSync`
  is used defensively — a stale/foreign `<projectId>` directory from a
  renamed project is possible in practice, same "which project dir is
  authoritative" ambiguity `workspace-code-intel.ts`/`workspace-setup.ts`
  never had to solve because they always ALREADY know `effectiveProjectId`
  at call time). Rather than re-deriving `effectiveProjectId` (which
  requires the user-level registry, home dir resolution, etc. — a much
  heavier dependency than this seam should carry), `readEnabledProviders`
  takes the FIRST `workspace.json` found (there is realistically always
  exactly 0 or 1 for a given `projectRoot`; a caller that already knows
  the exact path may pass `opts.workspaceJsonPath` to skip the scan
  entirely) and falls back to "no providers enabled" (empty list, never
  throws) when none is found or the file does not parse — matching
  every other best-effort resolution function in this codebase
  (`resolveMemoryBackend`, `readProvidersYaml`).
- **`engram` is reported by `buildProjectProviderRegistry`, not
  registered as a `ProviderClient`**: forcing engram's `MemoryBackend`
  into the `ProviderClient` shape (`supports()`/`invoke()`) would require
  either (a) a synthetic adapter with no real capability id to route to
  (engram maps to `memory.decisionRecall` in `providers.yaml`, but that
  capability is ALREADY served by `resolveMemoryBackend` directly via
  the `memory` MCP server kind — routing it a SECOND time through
  `invokeCapabilityLive`/`ProviderRegistry` would create two divergent
  resolution paths for the same capability), or (b) leaving `invoke()`
  unimplemented/throwing, violating the `ProviderClient` contract's
  own never-throw discipline. Reporting `engramEnabled: boolean` (backed
  by the SAME `resolveMemoryBackend` AB5 already built) gives callers
  (the new doctor check, docs) a uniform, truthful signal without a
  fake integration.
- **Doctor check is `warn`, never `fail`, for "enabled but not
  installed"**: mirrors AB4's `initializeCodeIntelIndexes`'s own
  discipline (a missing LOCAL dev tool is an environment fact, not a
  project defect) and `CC-*`'s own convention of reserving `fail` for
  genuine schema/model violations, not "an optional tool isn't on this
  particular machine."
- **AGENTS.md guidance heading kept as "Code intelligence tools"** (not
  renamed to something broader like "Provider tools") even though it now
  also covers the memory/engram line: AB4's existing tests
  (`canonical-agents-md.test.ts`) assert the literal string `'## Code
  intelligence tools'` in several places; renaming it would be a gratuitous
  breaking change to already-closed, passing tests for a purely cosmetic
  win. The engram line is appended under the SAME heading with its own
  one-line guidance ("recall past decisions/context before repeating
  work"), consistent with the existing terse, "when to reach for this"
  style.

## Implementation notes

**Files created:**
- `Axiom/packages/providers/src/selectable.ts` — `SELECTABLE_PROVIDER_IDS`,
  `isSelectableProviderId`, `isCodeIntelProviderId`.
- `Axiom/packages/providers/src/project-registry.ts` —
  `readEnabledProviders`, `buildProjectProviderRegistry`.
- `Axiom/packages/providers/tests/selectable.test.ts`.
- `Axiom/packages/providers/tests/project-registry.test.ts`.
- `Axiom/packages/doctor/tests/provider-selection.test.ts`.

**Files edited:**
- `Axiom/packages/providers/src/index.ts` — barrel exports.
- `Axiom/apps/cli/src/commands/tui.ts` — `providers` multi-select step;
  `buildWorkspaceSetupSpecFromWizard` parses `enabledProviders`.
- `Axiom/apps/cli/src/commands/workspace-setup.ts` —
  `WorkspaceSetupSpec.enabledProviders`, `workspace.json`'s `providers`
  field, effective code-intel list precedence, new best-effort engram
  probe step.
- `Axiom/apps/cli/src/commands/configure.ts` — `--providers <csv>` flag,
  `ConfigureArgs.providers`, `workspace.json` merge-write.
- `Axiom/packages/document-bootstrap/src/canonical-agents-md.ts` —
  `enabledMemoryProvider` guidance line.
- `Axiom/packages/doctor/src/checks.ts` / `index.ts` / `package.json` /
  `tsconfig.json` — `runProviderSelectionChecks` (`PS-001`) +
  `runProviderSelectionChecksAsync` (exposed, not wired into
  `runDoctorChecks`'s sync tree), new `@axiom/providers` dependency.
- `Axiom/apps/cli/package.json` / `tsconfig.json` — explicit
  `@axiom/providers` dependency (previously only transitive).
- `Axiom/apps/cli/tests/tui.test.ts` — updated the 2 existing end-to-end
  wizard scenarios (Scenario 6/7) to account for the new `providers` step
  (one extra scripted input line each; behavior otherwise unchanged).
- `Axiom/apps/cli/tests/workspace-setup.test.ts` — new Scenario (k) (6
  tests): `enabledProviders` persistence, empty-selection, omitted-both-
  fields default, precedence over `enabledCodeIntelProviders`,
  engram-only not triggering code-intel warnings, AGENTS.md engram
  guidance.
- `Axiom/apps/cli/tests/configure.test.ts` — new Scenario 5 (4 tests):
  `--providers`/`args.providers` persistence, empty-selection,
  omitted-field no-op, merge-over-existing-`workspace.json`.
- `Axiom/packages/document-bootstrap/tests/canonical-agents-md.test.ts` —
  new Scenario 1c (4 tests): `enabledMemoryProvider` guidance line.
- `Axiom/packages/providers/tests/selectable.test.ts` (new, 7 tests).
- `Axiom/packages/providers/tests/project-registry.test.ts` (new, 13 tests).
- `Axiom/packages/doctor/tests/provider-selection.test.ts` (new, 8 tests).

## Validation

- `npm run build` (`tsc -b`, from `Axiom/`): clean, no errors.
- `npx vitest run apps/cli packages/providers packages/tui packages/doctor`:
  ```
  Test Files  92 passed (92)
       Tests  902 passed (902)
  ```
- `npx vitest run` (full suite, parallel — default config):
  ```
  Test Files  189 passed (189)
       Tests  1978 passed (1978)
  ```
- `npx vitest run --no-file-parallelism` (full suite, serial):
  ```
  Test Files  189 passed (189)
       Tests  1978 passed (1978)
  ```
  0 regressions in either mode. Baseline (AB5 closing state) was
  `186/1936`; delta is `+3` new test files
  (`packages/providers/tests/{selectable,project-registry}.test.ts`,
  `packages/doctor/tests/provider-selection.test.ts`), `+42` new tests
  (all new, all passing), across new files plus new scenarios added to
  `apps/cli/tests/{tui,workspace-setup,configure}.test.ts` and
  `packages/document-bootstrap/tests/canonical-agents-md.test.ts`.
- `npm run doctor` (self-dogfood): `42/55 OK · 0 FALLO · 3 ADVERTENCIA ·
  10 OMITIDO · Resultado: PASS` — one more check than AB5's closing state
  (`41/54`) thanks to the new `PS-001`, reporting `pass` (no providers
  enabled in this repo's own `workspace.json`); still 0 FALLO.

## Result

Provider selection is now explicit end-to-end. The TUI wizard's new
`providers` multi-select step (`codegraph`/`serena`/`graphify`/`engram`,
none preselected) flows through `buildWorkspaceSetupSpecFromWizard` into
`WorkspaceSetupSpec.enabledProviders`, which `runWorkspaceSetup` persists
into `.axiom-state/<projectId>/workspace.json`'s new `providers` field
and uses to drive the AB4 code-intel init hook plus a new best-effort
engram probe — never touching `providers.yaml`'s schema-locked,
doctor-validated 7-provider closed set (a deliberate, heavily-documented
deviation from the brief's literal wording, made necessary by
`CC-001`/`ProviderRegistrySchema`). `axiom configure --providers <csv>`
offers the same persistence outside the wizard flow. `@axiom/providers`
gained `buildProjectProviderRegistry`, the single entry point that reads
a project's selection and returns a `ProviderRegistry` with exactly the
enabled LOCAL code-intel clients registered (plus always-on
`filesystem`); it is consumed by a new doctor check (`PS-001`) that
reports enabled-vs-installed status per provider, `warn`-only (never
`fail`) for a missing local tool. `engram` is reported alongside the
code-intel providers via the same `resolveMemoryBackend` probe AB5 built,
rather than forced into a `ProviderClient` shape it does not naturally
fit. Full command/skill/agent-level wiring of
`buildProjectProviderRegistry` beyond the doctor check is noted as
follow-up, not required by this increment's acceptance criteria.

## General spec integration

Integrated into `Axiom.Spec/specs/00-08` by the batch's final
consolidation pass (autopilot orchestrator). Landed in:

- **06_Integraciones_y_Capacidades.md** — PRIMARY. New subsection
  "Selección de providers (project-scoped, no false-breadth)":
  `SELECTABLE_PROVIDER_IDS`, `buildProjectProviderRegistry`, the doctor
  `PS-001` check, and the deliberate "`providers.yaml` NOT trimmed"
  deviation (schema-locked closed 7-set vs. `workspace.json#providers`
  selection).
- **03_Modelo_Operativo_y_Datos.md** — added the `providers` field to
  `WorkspaceSetupRecord`/`workspace.json` and the provider-selection
  read path.
- **05_Interfaces_Operativas.md** — `axiom configure --providers <csv>`
  and the wizard `providers` multi-select step.
- **01_Requisitos_Funcionales.md** — RF-AXM-025 (provider selection
  half).
- **07_Gobierno_y_Seguridad.md** — LOCAL-only posture + `PS-001` is
  `warn` not `fail`.
- **08_Glosario.md** — new terms: provider selection
  (`workspace.json#providers`), `buildProjectProviderRegistry`.

The still-open note that `invokeCapabilityLive`/`ProviderRegistry` has
no in-repo consumer beyond tests and `PS-001` was carried into 06 as a
"cableado pendiente" follow-up. The full command/skill/agent routing was
NOT taken up by AB7 (adapters-depth), whose own scope confirmed it is a
separate future increment.
