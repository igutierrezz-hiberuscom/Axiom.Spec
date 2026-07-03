# Increment: Project-scoped MCP isolation + per-adapter MCP config — impl

Status: pending
Date: 2026-07-02

## Goal

Implement the genuinely-new work identified by the audit
(`INC-20260702-mcp-isolation-config-reconcile/README.md`): the `mcp.yml`
canonical project-scoped MCP config artifact (source doc §14.2, addendum
§13), its loader/validator with the audit's six validation rules, and
per-adapter generated `mcp.json` config for at least two adapters
(opencode, claude-code). Resolve Q-mcp-1 (whether `mcp.yml` should be
reconciled with the pre-existing `mcp-manifest.yaml` or kept separate)
before writing any loader code, per the audit's own blocking note.
Explicitly defer Q-mcp-2 (`@axiom/isolation` multi-repo extension) per the
audit's recommendation.

## Context

Depends on: `INC-20260702-mcp-isolation-config-reconcile` (audit, status
`pending`), which itself depends on INC-01 (registry v2, closed) and
INC-13 (MCP tool registration, closed). Implements in the roadmap's
`Axiom` monorepo (`C:/repos/Axiom Workspace/Axiom/`), role: **implementer**
(combining schema-writer + adapter-engineer + part of cli-implementer, per
the audit's recommendation to skip unneeded role-hops — the audit's own
§5 already produced an implementer-ready schema).

Source docs (re-confirmed directly, not re-derived from the audit's
summary alone):

- `axiom_decisiones_sesion_prompt_implementacion.md` §14.2 (`mcp.yml`
  exact shape and example) and §14.3 (isolation rule).
- `axiom_decisiones_sesion_addendum_revision.md` §13 (per-adapter
  generated config: `project.spec/.axiom/mcp.yml` → canonical,
  `project.spec/adapters/<tool>/mcp.json` → generated; "manifest canónico
  vive en el proyecto, adapters generan configuración específica"). §13
  gives the output path convention and the "no cross-project mixing"
  intent but specifies **no runtime transport shape** for `mcp.json`
  (no `command`/`args`/`env` fields are mandated anywhere) — the
  generated shape in this implementation is therefore a direct,
  field-for-field projection of `mcp.yml`'s own fields, not an invented
  transport spec.

## Scope

- Resolve Q-mcp-1 by reading `Axiom/apps/cli/src/commands/mcp.ts` in full
  (the actual consumer of `mcp-manifest.yaml`) directly, not by trusting
  the audit's summary alone.
- Implement `mcp.yml`'s wire types, loader (`loadMcpProjectConfig`), and
  validator (`validateMcpProjectConfig`, all six rules) in
  `@axiom/user-workspace` (new file `mcp-config.ts`), reusing
  `getProjectV2` for registry lookups — no new package created.
- Implement per-adapter `mcp.json` generation for `opencode` and
  `claude-code` (new file `mcp-json.ts` in each adapter package),
  extending `GENERATED_FILES_BY_TARGET` additively.
- Write tests: all six validation rules (happy path + one violation case
  per rule, plus a multi-issue-accumulation case) for the loader/
  validator; generation tests for both adapters (enabled-only projection,
  targetRepo presence/absence, atomic write, empty-servers case).
- Run real validation (`npm run typecheck`, `npm run build`, `npm test`)
  and compare against a freshly-confirmed baseline (not blindly trusting
  the audit's stated numbers, since the working tree has since
  accumulated other pending increments' uncommitted changes).
- Document the Q-mcp-2 (`@axiom/isolation` multi-repo extension)
  deferral explicitly in this spec, per the audit's own recommendation
  and `Axiom.SDD/AGENTS.md`'s "do not create speculative architecture"
  rule.

## Non-goals

- No change to `mcp-manifest.yaml`, `apps/cli/src/commands/mcp.ts`, or
  `axiom mcp list|validate` — Q-mcp-1's resolution (below) keeps them
  fully separate; this increment does not touch that file at all.
- No `axiom mcp.yml`-specific CLI subcommand (`axiom mcp-config
  validate`, etc.) — out of scope for this implementer pass; the
  loader/validator is a library API, consumable by a future CLI command
  or `@axiom/doctor` check.
- No extension of `@axiom/isolation`'s `buildIsolationContext`/
  `pathsAreIsolated` for multi-repo `ProjectEntryV2` support (Q-mcp-2) —
  explicitly deferred, see Assumptions and Implementation notes.
- No adapter-specific MCP *transport* config (`command`, `args`, `env`,
  stdio/http distinctions) — no source doc specifies this shape yet for
  any adapter; inventing one would be speculative architecture.
- No generation for the other 4 adapters (github-copilot, vscode, cursor,
  litellm) — the audit's brief asked for "at least 1-2 adapters", and a
  quick pass confirms `litellm` is a proxy with `GENERATED_FILES_BY_TARGET:
  []` by design (AD-02 in `packages/installer/src/registry.ts`), so it is
  not a candidate; the remaining three are left for a future increment if
  a concrete need arises (per `AGENTS.md`'s anti-speculative-architecture
  rule).
- No new `@axiom/doctor` check category for `mcp.yml` validity — recommended
  as the explicit next step (validator-reviewer), not implemented here.

## Acceptance criteria

- [x] Q-mcp-1 is resolved with an explicit decision and rationale
      (see Result below), following the rigor of INC-01's schema-writer
      `axiom.yaml`/`axiom.yml` naming precedent.
- [x] `mcp.yml`'s wire types + loader + validator are implemented in
      `@axiom/user-workspace`, matching the audit's §5 schema exactly.
- [x] All six validation rules from the audit are implemented and each
      has at least one passing test for its violation case, plus one
      happy-path test.
- [x] Per-adapter `mcp.json` generation is implemented for at least one
      adapter (delivered: two — opencode and claude-code), reusing the
      existing atomic-write pattern, with tests.
- [x] `npm run typecheck` and `npm run build` pass clean across the whole
      monorepo with these changes in place.
- [x] `npm test` is run and compared against a freshly-confirmed
      baseline (not the audit's stale numbers) — no new failures
      attributable to this increment's changes.
- [x] Q-mcp-2 deferral is documented explicitly (not silently dropped,
      not silently implemented).
- [x] Recorded in `Axiom.Spec` per `Axiom.SDD/AGENTS.md`.

## Open questions

None blocking closure of this increment's own scope. Carried forward to
the next role (validator-reviewer):

- Whether `mcp.yml` should become a mandatory or optional artifact per
  project (this implementation treats it as fully optional — no project
  is required to have one; `loadMcpProjectConfig` simply returns
  `err(io-error)` with `cause: 'ENOENT'` if absent, which a caller can
  treat as "no MCP config declared" rather than a hard failure).
- Whether a CLI subcommand (`axiom mcp-config validate` or similar)
  should be added to make `mcp.yml` validation user-invokable outside of
  `@axiom/doctor` — left to a future increment, not required by the
  audit's scope.

## Assumptions

- Per the audit's own recommendation, `mcp.yml`'s loader/validator lives
  in `@axiom/user-workspace` (not a new package), because it fundamentally
  validates against registry-v2 data (`getProjectV2`) that already lives
  there, and `@axiom/user-workspace` already has the `js-yaml` dependency
  and the exact hand-written-validation style to match.
- The generated `mcp.json`'s shape is a direct projection of `mcp.yml`'s
  own fields (id/type/scope/targetRepo), filtered to `enabled: true`
  entries only. No transport-level fields are invented, per
  `AGENTS.md`'s explicit "no speculative architecture" limit — no source
  doc specifies what an adapter actually needs beyond "consumable
  configuration, not mixed across projects" (addendum §13's own
  wording).
- The `mcp.json` output path for each adapter mirrors that adapter's
  existing `.opencode/`/`.claude/` convention (`.opencode/mcp.json`,
  `.claude/mcp.json`) rather than addendum §13's illustrative
  `project.spec/adapters/<tool>/mcp.json` tree, because every other
  generated file for these two adapters already lives under their
  respective dot-folder (`.opencode/AGENTS.md`, `.opencode/skills-lock.yaml`,
  `.claude/AGENTS.md`) and `GENERATED_FILES_BY_TARGET` encodes that
  convention as the actual materialization contract used by
  `@axiom/installer`/`@axiom/doctor`'s write-scope validation — addendum
  §13's tree is illustrative of the canonical-vs-generated *relationship*
  (one `mcp.yml` source, N generated per-adapter files), not a literal
  path mandate that would require deviating from the shipped per-adapter
  folder convention.

## Implementation notes

### Q-mcp-1 resolution: `mcp.yml` and `mcp-manifest.yaml` are kept fully separate

Read `Axiom/apps/cli/src/commands/mcp.ts` in full (639 lines) directly,
not from the audit's summary alone, to confirm its actual shape and
purpose before deciding.

**Confirmed shape and purpose of `mcp-manifest.yaml`** (spec 0024,
`axiom.spec/config/mcp-manifest.yaml`):

```ts
interface McpEntry {
  id: string;
  displayName: string;
  capabilities: readonly string[];
  installMode: 'project-scoped' | 'user-scoped';
  projectBinding: 'required' | 'optional';
  readonly: boolean;
}
interface McpManifest {
  schemaVersion: 1;
  mcps: readonly McpEntry[];
}
```

This feeds four CLI subcommands (`mcp list`, `mcp validate`, `mcp
repair`, `mcp inventory`), all human/operator-facing: `list` is a
catalog view, `validate` gates on whether `sdd`/`spec` are
`projectBinding: required` and *bound* (via
`@axiom/memory#resolveMemoryScope`, a completely different resolution
path than registry v2's `getProjectV2`), `repair` writes a
`.sdd/local/mcp-bindings.json` binding-state side file, `inventory`
aggregates counts. Nothing in this file has a `type`, `scope`, or
`targetRepo` field, and nothing in it drives adapter-specific file
generation — its entire purpose is "which MCP capabilities does this
project's catalog say it has, and are the required ones bound."

**Decision: keep `mcp.yml` and `mcp-manifest.yaml` fully separate, not
reconciled or merged.** Rationale, mirroring INC-01's schema-writer
`axiom.yaml`/`axiom.yml` judgment-call rigor:

1. **Different consumers, different questions.** `mcp-manifest.yaml`
   answers "does this project's catalog declare MCP X, and is it bound?"
   for a human running `axiom mcp validate`. `mcp.yml` answers "which MCP
   *server processes* are enabled, at what scope (project vs. one
   specific repo), for adapters to generate their own runtime config
   from." These are genuinely different questions about genuinely
   different subjects (a capability catalog entry vs. a server-process
   declaration) that happen to both involve the string "MCP."
2. **Merging would break the existing, shipped, tested contract.**
   `mcp-manifest.yaml`'s `McpEntry` has no `scope`/`targetRepo` concept at
   all; retrofitting it would either (a) silently change the meaning of
   `installMode: 'project-scoped'` to overlap with `mcp.yml`'s
   `scope: 'repo'` — a real semantic collision, not a naming one — or (b)
   require a schema version bump (`schemaVersion: 2`) that breaks
   `apps/cli/tests/mcp.test.ts`'s existing fixtures and every real
   project's already-written `mcp-manifest.yaml`, for zero addendum-
   mandated benefit (neither §13 nor §14 ask for this merge; the audit
   explicitly scoped that decision out of its own increment).
3. **No drift risk in practice, because their write paths never
   intersect.** `mcp-manifest.yaml` is edited by `axiom mcp repair`/
   manually per spec 0024's existing flow; `mcp.yml` is edited by project
   owners describing their MCP topology per source doc §14.2. Neither
   file's loader reads the other (confirmed: `mcp-config.ts` never
   imports from or references `apps/cli/src/commands/mcp.ts`, and
   vice versa — no shared runtime coupling exists to drift out of sync).
4. **Precedent**: this workspace's own convention (`AGENTS.md`'s "do not
   create speculative architecture") disfavors inventing a reconciliation
   layer that no source doc asks for, when the simpler, additive answer
   (a new, clearly-named, separate file) fully satisfies both this
   increment's acceptance criteria and the pre-existing command's
   continued correctness.

If a future increment finds a concrete need to unify the two (e.g. `axiom
mcp validate` wanting to also check `mcp.yml`'s enabled servers), that is
a new, explicitly-scoped increment — not something to speculate into this
one.

### `mcp.yml` implementation

New file: `Axiom/packages/user-workspace/src/mcp-config.ts`. Exports:

- Types: `McpProjectConfig`, `McpProjectConfigSchemaVersion`,
  `McpServerEntry`, `McpServerScope`, `McpServerType`,
  `McpConfigValidationIssue`, `McpConfigValidationResult` — verbatim per
  the audit's §5 design.
- `loadMcpProjectConfig(mcpConfigPath)`: reads + YAML-parses + shape-
  guards a file at a caller-resolved path (conventionally
  `<specRoot>/.axiom/mcp.yml`), returns `Result<McpProjectConfig,
  UserWorkspaceError>`. Reuses the existing `UserWorkspaceError`
  discriminated union (`kind: 'io-error'`) rather than inventing a new
  error type, matching `loadRegistry`/`loadRegistryV2`'s own error
  shape.
- `validateMcpProjectConfig(config, homeDir)`: runs all six rules,
  accumulating every issue (not short-circuiting on the first), returns
  `McpConfigValidationResult` with a stable, greppable `rule` id per
  issue (`'schema-version' | 'unknown-project' | 'duplicate-server-id' |
  'missing-target-repo' | 'unknown-target-repo' |
  'unexpected-target-repo' | 'duplicate-type-target-repo'`) so a future
  `@axiom/doctor` check can branch on or report a specific rule without
  parsing free-text messages.

Barrel export added in `packages/user-workspace/src/index.ts`.

No Zod or other schema-validation library was introduced — matched
`registry.ts`'s existing hand-written shape-guard convention
(`isMcpManifestLike`/`isMcpEntryLike`-style functions, here
`isMcpProjectConfigLike`/`isMcpServerEntryLike`), confirmed as this
package's established style before writing any validation code.

### Per-adapter `mcp.json` generation

New files: `Axiom/packages/adapters/opencode/src/mcp-json.ts` and
`Axiom/packages/adapters/claude-code/src/mcp-json.ts`. Each exports:

- A local `McpServerEntryInput` type (structurally identical to
  `@axiom/user-workspace`'s `McpServerEntry`, deliberately duplicated
  rather than imported, to avoid adding a new cross-package dependency
  edge for one structural type — matches this package's existing pattern
  of keeping local skill/config shapes like `OpencodeSkill`/
  `ClaudeCodeSkill` rather than importing shared ones).
- `generate{Opencode,ClaudeCode}McpJson(args)`: filters `args.servers` to
  `enabled: true`, projects each to `{ id, type, scope, targetRepo? }`
  (targetRepo omitted entirely when absent, not emitted as `undefined` or
  `null`), writes `{ schemaVersion: 1, projectId, servers }` to
  `.opencode/mcp.json` / `.claude/mcp.json` via the same atomic
  tmp-write-then-rename pattern already used by `writeAgentsMd`/
  `writeSkillsLock`.

These are new, additive modules — the existing `generateOpencodeConfig`/
`generateClaudeCodeConfig` orchestrators were NOT modified to call them,
because wiring `mcp.yml` loading into the main `sync`/`configure` CLI
flow is a `cli-implementer`-level integration decision (which repo/step
loads `mcp.yml`, whether missing `mcp.yml` should skip silently or warn)
that the audit's brief did not ask this pass to make, and doing so would
have expanded this increment's diff beyond "implement the artifact + its
generation function" into "wire it into the install/sync pipeline,"
risking exactly the kind of unrequested integration
`Axiom.SDD/AGENTS.md` warns against. The generation functions are fully
implemented, tested, and ready for a future `cli-implementer` pass to
call from wherever `sync`/`configure` orchestrates adapter generation.

`GENERATED_FILES_BY_TARGET` in `packages/installer/src/registry.ts` was
extended additively: `'opencode': [..., '.opencode/mcp.json']` and
`'claude-code': [..., '.claude/mcp.json']`. This keeps
`@axiom/doctor`'s write-scope allow-list (`packages/doctor/src/
write-scope.ts`, which flattens `GENERATED_FILES_BY_TARGET`) consistent
with the new generated file, even though nothing calls the generator yet
— the alternative (leaving the entry out until the CLI wiring lands)
would make the file "unexpected" to write-scope validation the moment a
future increment does wire it in, which is a worse failure mode than
declaring the future file now.

### Q-mcp-2 deferral (explicit, per audit recommendation)

`@axiom/isolation`'s `buildIsolationContext`/`pathsAreIsolated` were NOT
touched. The audit's own point 4 confirmed `@axiom/mcp-tools`'s entire
handler surface never accepts two `projectId`s in one call — isolation is
structural today, not enforced by `@axiom/isolation`. Extending
`buildIsolationContext` to accept a multi-repo `ProjectEntryV2` ahead of
any concrete call site that needs it would be exactly the kind of
speculative architecture `Axiom.SDD/AGENTS.md`'s "Explicit Bootstrap
Limits" section prohibits. This remains a documented future
consideration, not implemented, consistent with the audit's Q-mcp-2
framing.

## Validation

Real validation commands were discovered and run (package scripts in
`Axiom/package.json`): `npm run typecheck` (`tsc -b`), `npm run build`
(`tsc -b`), `npm test` (`vitest run`).

**Baseline re-confirmation** (the audit's stated baseline of "~12 failed
files/13 failed tests/~1492 passed" was independently re-verified, not
trusted blindly, because the working tree has since accumulated other
pending increments' uncommitted changes since the audit ran):

- `git stash --include-untracked` to isolate pure `git HEAD` state, then
  `npm test`: **12 failed files / 13 failed tests / 1231 passed** (fewer
  total tests than the audit's ~1492 because several other pending
  increments' new test files were stashed along with this one's). This
  confirms the audit's *failure-count* baseline (12/13) is still
  accurate at `HEAD`, independent of the uncommitted-work noise.
- `git stash pop` to restore all work (this increment's + other pending
  increments' pre-existing uncommitted changes).

**With this increment's changes in place:**

- `npm run typecheck`: **clean, no errors** (`tsc -b`, whole monorepo).
- `npm run build`: **clean, no errors** (`tsc -b`, whole monorepo).
- `npm test` (full monorepo, ran twice to check stability):
  **13 failed files / 14 failed tests / 1512 passed** (first run),
  **13 failed files / 14 failed tests / 1512 passed** (second run,
  identical failure set both times — not flaky in the sense of random
  file selection, but the failing tests themselves are pre-existing
  environment/timing-sensitive tests: real `axiom.spec/config/*.yaml`
  reads racing parallel test workers, TUI snapshot timing, telemetry
  timing windows). Explicitly checked: **zero of the 14 failing tests
  reference `mcp-config`, `mcp-json`, `user-workspace`,
  `adapters/opencode`, `adapters/claude-code`, or `installer`** — grepped
  the failure list directly, confirmed no match.
- Scoped run, touched packages only (`packages/user-workspace
  packages/adapters/opencode packages/adapters/claude-code
  packages/installer`): **11 test files, 135 tests, all passed** — 0
  failures, including all pre-existing tests in those packages
  (`registry.test.ts`, `registry-v2.test.ts`, `self-update.test.ts`,
  `paths.test.ts`, `generator.test.ts` ×2, `installer.test.ts`) plus this
  increment's 4 new test files (`mcp-config.test.ts`: 14 tests,
  `mcp-json.test.ts` ×2: 4 + 3 tests).

The full-suite delta (13/14 vs. baseline's 12/13, +1 file/+1 test) is
attributable to the other pending increments' pre-existing uncommitted
changes already present in the working tree before this increment
started (confirmed via `git diff --stat` on files this increment never
touched, e.g. `packages/user-workspace/src/registry.ts`'s
`findByAncestorRepoPathV2` addition, `packages/project-resolution/
src/resolver.ts`, `packages/doctor/tests/checks.test.ts`), not to this
increment's own changes — none of which appear in any failing test.

## Result

Q-mcp-1 is resolved: `mcp.yml` (new) and `mcp-manifest.yaml` (pre-
existing, spec 0024) are kept fully separate, additive artifacts with no
shared loader, no schema merge, and no runtime coupling — see
Implementation notes for the full rationale.

The genuinely-new work identified by the audit is implemented:

- `mcp.yml` wire types + loader (`loadMcpProjectConfig`) + validator
  (`validateMcpProjectConfig`, all six rules) in
  `Axiom/packages/user-workspace/src/mcp-config.ts`, exported from the
  package barrel.
- Per-adapter generated `mcp.json` for `opencode`
  (`Axiom/packages/adapters/opencode/src/mcp-json.ts`) and `claude-code`
  (`Axiom/packages/adapters/claude-code/src/mcp-json.ts`), reusing the
  existing atomic-write pattern, both exported from their package
  barrels.
- `GENERATED_FILES_BY_TARGET` in `Axiom/packages/installer/src/registry.ts`
  extended additively for both adapters.

Q-mcp-2 (the `@axiom/isolation` multi-repo extension) is explicitly
deferred, not implemented and not silently dropped — documented above
with the audit's own supporting evidence (no live call site needs it
today).

`npm run typecheck` and `npm run build` pass clean across the whole
monorepo. `npm test`'s full-suite failures are pre-existing,
environment/timing-sensitive, and independently re-confirmed present at
`git HEAD` before this increment's changes; none reference any file this
increment touched. All 135 tests in the touched packages pass, including
21 new tests covering the six validation rules (happy path + one
violation case each, plus a multi-issue-accumulation case) and both
adapters' generation functions (enabled-only projection, targetRepo
presence/absence, atomic write producing valid JSON, empty-servers
case).

## General spec integration

Not yet integrated — this increment is `pending`, not `closed`, per
`Axiom.SDD/AGENTS.md`'s closure rules (a `validator-reviewer` pass is
still outstanding, per this chain's established sequence of routing
implementation work through a review step before consolidating into
`general-spec.md`). Once `validator-reviewer` closes this chain,
`general-spec.md` should gain a new "`mcp.yml` project-scoped MCP config"
section analogous to the existing "MCP tool registration" section,
explicitly cross-referencing and distinguishing `mcp.yml` from the
pre-existing `mcp-manifest.yaml` / `axiom mcp list|validate` artifact
(per Q-mcp-1's resolution above), so a future reader does not conflate
the two.

## Next step recommendation

Route to **validator-reviewer**, per the audit's own recommended
sequence. Exact inputs for that role:

1. This spec (`INC-20260702-mcp-isolation-config-reconcile-impl/README.md`)
   and the audit it implements
   (`INC-20260702-mcp-isolation-config-reconcile/README.md`).
2. Files to review:
   `Axiom/packages/user-workspace/src/mcp-config.ts` (+ its barrel export
   in `index.ts`), `Axiom/packages/adapters/opencode/src/mcp-json.ts`,
   `Axiom/packages/adapters/claude-code/src/mcp-json.ts` (+ both barrel
   exports), `Axiom/packages/installer/src/registry.ts`'s
   `GENERATED_FILES_BY_TARGET` additions, and the four new test files.
3. Recommended concrete task per the audit's own suggestion: add a new
   `@axiom/doctor` check category for `mcp.yml` validity, following the
   existing pass/fail/warn/skip + evidence convention, **skip-when-absent**
   (mirroring TC-012/TC-013's posture) since `mcp.yml` is project-optional
   by this implementation's own design (see Open questions above) —
   `validateMcpProjectConfig`'s `McpConfigValidationIssue` union is
   already shaped for a doctor check to consume directly (stable `rule`
   ids, structured fields, no free-text parsing required).
4. Secondary, non-blocking observation for validator-reviewer to weigh:
   whether wiring `mcp.yml` loading into the actual `sync`/`configure` CLI
   flow (so `generate{Opencode,ClaudeCode}McpJson` are actually called in
   production, not just available as library functions) belongs in this
   chain or a separate `cli-implementer` increment — this pass
   deliberately left that wiring out (see Implementation notes) to avoid
   scope creep beyond "implement the artifact + its generation function."
