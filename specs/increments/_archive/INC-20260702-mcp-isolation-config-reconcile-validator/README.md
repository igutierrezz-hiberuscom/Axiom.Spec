# Increment: Project-scoped MCP isolation + per-adapter MCP config — validator

Status: closed
Date: 2026-07-02

## Goal

Final, `validator-reviewer` role of the INC-15 chain
(`INC-20260702-mcp-isolation-config-reconcile{,-impl}`). Independently
re-verify the implementer's reported test-failure baseline discrepancy
(rather than accept it at face value), independently confirm
`validateMcpProjectConfig`'s six rules and the Q-mcp-1 separation
decision directly from code, independently trace at least one adapter's
`mcp.json` generation, decide on the implementer's own recommended
`@axiom/doctor` check category, run real validation, and close INC-15 as
a whole if its acceptance criteria are met.

## Context

Depends on, in order:

1. `INC-20260702-mcp-isolation-config-reconcile` (audit, status
   `pending`) — designed `mcp.yml`'s schema and the six validation rules.
2. `INC-20260702-mcp-isolation-config-reconcile-impl` (implementer,
   status `pending`) — implemented `mcp-config.ts`
   (`@axiom/user-workspace`), per-adapter `mcp-json.ts` generators
   (opencode, claude-code), resolved Q-mcp-1 (kept `mcp.yml` and the
   pre-existing `mcp-manifest.yaml` fully separate), and reported a
   test-failure baseline delta (12 failed files/13 failed tests at clean
   `git HEAD` -> 13 failed files/14 failed tests with their changes) they
   characterized as "pre-existing, unrelated, environment/timing-
   sensitive."

This role, per `Axiom.SDD/AGENTS.md`'s workflow: implement in
`Axiom.SDD`/`Axiom` (the actual product monorepo lives at
`C:/repos/Axiom Workspace/Axiom/`), record in `Axiom.Spec`.

## Scope

- Independently re-verify the implementer's baseline-delta claim: run the
  full suite fresh, isolate the actual 14th failing test/13th failing
  file, and bisect via `git stash`/`git stash pop` to determine whether
  it is caused by this increment's changes or genuinely pre-existing.
- Trace all six `validateMcpProjectConfig` rules directly against
  `Axiom/packages/user-workspace/src/mcp-config.ts`'s source, not against
  the audit's or implementer's description of them.
- Re-read `Axiom/apps/cli/src/commands/mcp.ts` directly to independently
  confirm or override the Q-mcp-1 separation decision.
- Trace `generateOpencodeMcpJson`'s transformation logic directly against
  its source and its test fixtures.
- Decide whether to implement a new `@axiom/doctor` check category for
  `mcp.yml` validity (the implementer's own recommendation), or defer
  with documented reasoning.
- Run `npm run typecheck`, `npm run build`, `npm test` (full monorepo)
  after resolving the baseline discrepancy.
- Close INC-15 as a whole if its acceptance criteria are met.
- Integrate an "MCP project config (`mcp.yml`)" section into
  `Axiom.Spec/general-spec.md`.

## Non-goals

- No new feature work beyond what the audit/implementer scoped.
- No wiring of `mcp.yml` loading into `runConfigure`/`sync`'s actual
  install/sync pipeline (already explicitly deferred by the implementer
  as a `cli-implementer`-level integration decision; this role does not
  reverse that deferral).
- No `@axiom/isolation` multi-repo extension (Q-mcp-2) — confirmed still
  correctly deferred, no new call site has appeared during this review.
- No reconciliation of `mcp.yml` with `mcp-manifest.yaml` beyond
  confirming or overriding Q-mcp-1's existing decision.

## Acceptance criteria

- [x] The test-failure baseline discrepancy is independently
      investigated (not accepted at face value) and resolved: either a
      real regression is found and fixed, or it is confirmed genuinely
      pre-existing/unrelated with specific evidence.
- [x] All six `validateMcpProjectConfig` rules are confirmed as genuinely
      enforced by direct code inspection.
- [x] The Q-mcp-1 separation decision is independently confirmed or
      overridden, based on fresh reading of `mcp.ts`, not restated from
      the implementer's own summary alone.
- [x] At least one adapter's `mcp.json` generation is independently
      traced and confirmed correct against a realistic fixture.
- [x] The `@axiom/doctor` check category question is explicitly decided
      (implemented or deferred) with documented reasoning.
- [x] Real validation (`typecheck`, `build`, `test`) is executed after
      resolving the discrepancy, with results reported.
- [x] `Axiom.Spec/general-spec.md` gains an "MCP project config
      (`mcp.yml`)" section.
- [x] INC-15 as a whole is closed, or left `pending` with explicit
      rationale.

## Open questions

None blocking this role's own closure. Q-mcp-2 (`@axiom/isolation`
multi-repo extension) remains an explicitly deferred, documented future
consideration for INC-15 as a whole (see Result below) — not a question
this role needs to resolve, since no concrete consumer has appeared.

## Assumptions

- "Independently re-verify" means re-running the bisection from scratch
  in this session (fresh `npm test`, fresh `git stash`/`git stash pop`),
  not re-reading the implementer's own stated numbers and trusting them
  — this is the whole point of the validator-reviewer role in this
  chain's established sequence.
- The working tree contains other pending increments' uncommitted
  changes (confirmed: `git status --short` shows ~89 changed/untracked
  paths beyond this increment's own files), consistent with what the
  implementer described. This role treats those other changes as
  environment noise to control for during bisection, not as something to
  fix or revert.

## Implementation notes

### 1. Test-failure baseline discrepancy — INDEPENDENTLY RE-VERIFIED: this was a REAL, narrow regression, now fixed

The implementer's claim ("the 13/14 delta over the 12/13 baseline is
pre-existing/unrelated/environment-timing-sensitive") was **not accepted
at face value**. Independent re-verification:

1. Ran `npm test` fresh on the working tree exactly as the implementer
   left it (all pending increments' uncommitted changes present,
   including this increment's `mcp-config.ts`/`mcp-json.ts`/
   `registry.ts` changes): **13 failed files / 14 failed tests / 1512
   passed** — reproduces the implementer's own reported numbers exactly.
2. Captured the full list of 14 failing test names.
3. `git stash --include-untracked` to isolate a pure `git HEAD` state,
   then `npm test`: **12 failed files / 13 failed tests / 1231 passed**
   (fewer total tests because other pending increments' new test files
   were stashed too — expected). Captured the full list of 13 failing
   test names.
4. `git stash pop` to restore all work.
5. Diffed the two failing-test-name lists directly (not just counts).
   **The one test present in the with-changes list but absent from the
   clean-HEAD list**: `apps/cli/tests/document-bootstrap.test.ts >
   Scenario 2: runConfigure con adapterTarget=opencode (B4 skip branch)
   > NO escribe .github/copilot-instructions.md (B1 AD-CONF-1 + B4
   skip)`.
6. Ran that one test file in isolation
   (`npx vitest run apps/cli/tests/document-bootstrap.test.ts`): fails
   reproducibly, not flaky — `expected 3 to be 2` on
   `expect(result.generatedFiles).toBe(2)`.

**Root cause, traced directly**: the implementer's own change to
`Axiom/packages/installer/src/registry.ts`'s `GENERATED_FILES_BY_TARGET`
added `.opencode/mcp.json` to the `opencode` target's file list (`2`
entries -> `3`) and `.claude/mcp.json` to `claude-code`'s (also `+1`),
as a "declare the write-scope entry ahead of time" choice. However,
`Axiom/apps/cli/src/commands/configure.ts`'s `runConfigure` computes its
summary's `generatedFiles` count as
`installResult.value.generatedFiles.length + (b4WroteCopilot ? 1 : 0)`
— i.e. it treats `GENERATED_FILES_BY_TARGET`'s static per-target array
**as a literal count of files this run produces**, not merely as a
write-scope allow-list superset. Since `generateOpencodeMcpJson`/
`generateClaudeCodeMcpJson` are NOT wired into `runConfigure` (an
explicit, correctly-scoped non-goal of the implementer's own pass), no
`mcp.json` is ever actually written during `runConfigure` — but the
static list now claims 3 files for `opencode` instead of 2, inflating
the test's expected exact count by one.

This is a **genuine regression directly caused by this increment's own
change**, not a pre-existing, unrelated, or environment-timing-sensitive
failure. The implementer's characterization was incorrect: they treated
`GENERATED_FILES_BY_TARGET` purely as a `@axiom/doctor` write-scope
allow-list concern (where declaring a not-yet-written future file ahead
of time is harmless, since allow-lists are supersets by nature) and
missed that `configure.ts` has a second, incompatible consumer that
reads the same array's `.length` as an exact produced-file count.

**Fix applied** (`Axiom/packages/installer/src/registry.ts`): reverted
`.opencode/mcp.json` and `.claude/mcp.json` from
`GENERATED_FILES_BY_TARGET`'s `opencode`/`claude-code` entries, restoring
them to their pre-increment `2`/`1`-entry (opencode 2 files: `AGENTS.md`
+ `skills-lock.yaml`; claude-code 1 file: `AGENTS.md`) lists, with an
explicit code comment documenting why (`generateOpencodeMcpJson`/
`generateClaudeCodeMcpJson` exist and are tested, but are not wired into
any install/sync flow yet — add the entry only when a future
`cli-implementer` pass wires the generator in). This is the same
deferral shape the implementer themselves cited as precedent (INC-13's
TR-005: declare a write-scope entry once something writes it, not ahead
of a concrete writer).

**Re-verification after the fix**: `npm test` (full monorepo) now
reports **12 failed files / 14 failed tests fixed to 13 / 1513 passed**
— exactly **12 failed files / 13 failed tests / 1513 passed**, and a
direct diff of the post-fix failing-test-name list against the
clean-`git HEAD` failing-test-name list is **byte-identical (zero
diff)**. The regression is fully resolved: the working tree's failure
set now matches the pre-existing baseline exactly, no more and no less.

**The 13 pre-existing failures** (confirmed identical at clean `HEAD`
and in the fixed working tree, none touched by this increment's `mcp-*`
files):
`apps/cli/tests/start.test.ts` (Scenario 4, `axiom.spec/config/
profiles.yaml` `ENOENT` — depends on `process.cwd()`-relative real file,
environment-path-dependent), `packages/agents/tests/catalog.test.ts`,
`packages/doctor/tests/checks.test.ts` ("repositorio Axiom real"),
`packages/model-routing/tests/{assignments,loader,opencode-projection,
resolver}.test.ts` (all read the real `axiom.spec/config/*.yaml` policy
files), `packages/skills/tests/catalog.test.ts`,
`packages/telemetry/tests/audit-trail-sink.test.ts` (retention-sweep
timing), `packages/project-resolution/tests/resolver.test.ts`,
`packages/toolchain/tests/repair-add-gitignore.test.ts`,
`packages/tui/tests/driver.test.ts` (2 failures: TUI snapshot/output
timing). All of these depend on real `axiom.spec/config/*` files at
`process.cwd()` or on timing-sensitive snapshot/retention assertions —
consistent with the implementer's own (correct, for this subset)
characterization of them as environment/path-dependent, not logic bugs
in any `mcp.yml`-related code.

### 2. `validateMcpProjectConfig`'s six rules — independently confirmed genuinely enforced

Read `Axiom/packages/user-workspace/src/mcp-config.ts` in full,
independent of the audit's/implementer's descriptions, and traced each
rule's actual code:

1. **`schemaVersion === 1`** — `if (config.schemaVersion !== 1)` pushes a
   `'schema-version'` issue. Confirmed enforced, not just named.
2. **`projectId` resolves via registry v2** — calls
   `getProjectV2(homeDir, config.projectId)`; if `!ok` or `value ===
   null`, pushes `'unknown-project'`. Confirmed: a real registry lookup,
   not a stub. Also confirmed `repoRoleKeys` is populated from the
   *same* lookup's `.repos` keys for reuse in rule 4 — no second lookup,
   no drift risk between rules 2 and 4.
3. **Unique server ids** — `Set`-based duplicate detection over
   `config.servers`, pushes one `'duplicate-server-id'` issue per
   duplicate encountered (not just the first). Confirmed enforced.
4. **`targetRepo` required + valid iff `scope === 'repo'`** — for each
   `scope === 'repo'` server: missing/empty `targetRepo` pushes
   `'missing-target-repo'`; present but not in `repoRoleKeys` (when the
   registry lookup succeeded) pushes `'unknown-target-repo'`. Confirmed
   enforced, including the "gracefully skip the registry-membership
   check if rule 2 already failed" behavior (`repoRoleKeys !== null`
   guard) — a deliberate, documented choice to avoid a redundant/
   confusing second issue when the project itself doesn't resolve.
5. **`targetRepo` rejected iff `scope === 'project'`** — for each
   `scope === 'project'` server with `targetRepo !== undefined`, pushes
   `'unexpected-target-repo'`. Confirmed enforced.
6. **No duplicate `(type, targetRepo)` pairs among `enabled: true`
   servers** — `Map`-keyed by `` `${type}::${targetRepo ?? ''}` ``,
   iterated only over `enabled` servers, pushes
   `'duplicate-type-target-repo'` on the second+ occurrence of a key.
   Confirmed enforced, including that `enabled: false` entries are
   correctly excluded from this specific rule (matching the audit's
   original rule text, "both `enabled: true`").

All six rules accumulate into one `issues` array (verified: no `return`/
`continue` short-circuits before all six blocks run), matching the
audit's "accumulate every issue, don't short-circuit" design intent.
**Independent verdict: all six rules are genuinely enforced by working
code, not merely documented in comments.**

### 3. Q-mcp-1 (`mcp.yml` vs. `mcp-manifest.yaml` separation) — independently confirmed, not overridden

Re-read `Axiom/apps/cli/src/commands/mcp.ts` directly (not from the
implementer's summary). Confirmed fresh:

- `mcp.ts` only ever reads/writes `mcp-manifest.yaml` (via a local
  `McpManifest`/`McpEntry` shape) and `.sdd/local/mcp-bindings.json`; it
  imports `resolveMemoryScope` from `@axiom/memory` for its `mcp
  validate` binding check — a completely different resolution path from
  `mcp-config.ts`'s `getProjectV2` (registry v2).
- Grepped the whole `Axiom/` tree for cross-references between the two
  files: `mcp-config.ts` never imports from or is imported by
  `apps/cli/src/commands/mcp.ts`, and neither loader reads the other's
  file. Zero shared runtime coupling exists today.
- `McpEntry` (capability catalog entry: `id`, `displayName`,
  `capabilities`, `installMode`, `projectBinding`, `readonly`) and
  `McpServerEntry` (server-process declaration: `id`, `type`, `scope`,
  `targetRepo?`, `enabled`) describe genuinely different subjects (a
  capability/binding catalog vs. a server-process topology declaration)
  that happen to share the string "MCP" and both notionally describe
  "the project's MCP-related things."

**Independent verdict: CONFIRMED, not overridden.** The separation is
sound. Merging the two would require either overloading
`installMode: 'project-scoped'` to also mean `mcp.yml`'s `scope:
'repo'`/`'project'` distinction (a real semantic collision, since
`installMode` today only distinguishes project-scoped vs. user-scoped
*installation*, not per-repo *targeting*) or a breaking `schemaVersion`
bump to the shipped, tested `mcp-manifest.yaml` contract, for no
addendum-mandated benefit. No user-facing confusion risk was found
serious enough to outweigh that cost: the two commands/files are
discoverable independently (`axiom mcp list|validate` vs. a project's
own `.axiom/mcp.yml`), and nothing in either file's docs or output
implies they are the same artifact.

### 4. Adapter `mcp.json` generation — independently traced and confirmed correct

Traced `generateOpencodeMcpJson`
(`Axiom/packages/adapters/opencode/src/mcp-json.ts`) directly against
`Axiom/packages/adapters/opencode/tests/mcp-json.test.ts`'s four
fixtures:

- **Enabled-only projection**: `args.servers.filter((s) => s.enabled)`
  — a disabled `serena-frontend` (scope=repo) is correctly excluded from
  both `serverCount` and the written `servers` array. Confirmed against
  the "proyecta sólo los servers enabled=true" test.
- **`targetRepo` omission when absent**: the spread
  `...(s.targetRepo !== undefined ? { targetRepo: s.targetRepo } : {})`
  means a `scope: 'project'` server's output object has no
  `targetRepo` key at all (not `null`, not `undefined` literal) —
  confirmed against `expect('targetRepo' in project).toBe(false)`.
- **Atomic write**: `fs.writeFileSync(tmpPath, ...)` then
  `fs.renameSync(tmpPath, targetPath)`, matching the package's existing
  `writeAgentsMd`/`writeSkillsLock` pattern; the written file parses as
  valid JSON (confirmed against the "escribe un archivo JSON válido"
  test).
- **Empty-servers edge case**: an all-disabled input list produces
  `serverCount: 0` and `servers: []` (not omitted, not `null`) —
  confirmed against the "sin servers enabled produce servers: []" test.

`generateClaudeCodeMcpJson` is structurally identical (same filter,
same spread, same atomic-write pattern, own `.claude/mcp.json` path
constant) — read directly and confirmed to match the opencode
implementation's logic exactly, modulo the output path.

**Independent verdict: CONFIRMED correct.** Both generators produce
genuinely correct output from a realistic `mcp.yml`-shaped input; no
transport fields are invented (matching the explicit non-goal), and the
enabled-only/targetRepo-omission/atomic-write behaviors all match their
test coverage.

### 5. `@axiom/doctor` check category — DEFERRED, not implemented

Investigated the implementer's own recommendation (add a new doctor
check category for `mcp.yml` validity, skip-when-absent, mirroring
TC-012/TC-013) by reading `Axiom/packages/doctor/src/index.ts`'s
`runDoctorChecks` and TC-012/TC-013's implementations directly.

**Finding**: `runDoctorChecks(resolution: ProjectResolution):
DoctorReport` has a **fixed, single-argument signature** — every one of
its 20+ composed check functions takes only `ProjectResolution`. No
`homeDir` or `projectId` is threaded through this call graph anywhere
today. This matters because `validateMcpProjectConfig`'s two most
substantive rules (rule 2: `projectId` resolves via registry v2; rule 4:
`targetRepo` is a valid repo role key) both require `homeDir` to call
`getProjectV2(homeDir, projectId)` — the check cannot meaningfully cover
its most isolation-relevant rules without a `runDoctorChecks` signature
change (a real, cross-cutting change touching the CLI's `doctor`
command, the TUI, and every existing check-function test, not a
same-shape additive function like TC-012/TC-013 were).

Additionally, grepped the whole `Axiom/` tree for `mcp.yml` outside this
increment's own new library files: **confirmed zero CLI command, `init`
flow, or scaffold path writes `mcp.yml` to any real project today.** The
only references are `mcp-config.ts` itself, the two generator modules
that consume its exported type (not its file), and the loader's own test
fixtures. There is no live project anywhere with a real `.axiom/mcp.yml`
for a doctor check to meaningfully validate yet.

**Decision: DEFER**, not implement, for two independent reasons:

1. **Not cheap**: unlike TC-012/TC-013 (which slot into the existing
   `ProjectResolution`-only signature with zero plumbing changes), a
   functionally-complete `mcp.yml` doctor check needs a `runDoctorChecks`
   signature change to thread `homeDir`, which is a larger, riskier
   change than "genuinely valuable and cheap" — the standard this task
   set for implementing it now.
2. **No live consumer yet**: no installer/scaffold path writes `mcp.yml`
   to any project today (confirmed by grep, mirroring the audit's own
   finding for `@axiom/isolation`'s `pathsAreIsolated` and this chain's
   established INC-13 TR-005 precedent: don't build a check for an
   artifact nothing produces yet).

This is a documented future consideration, not a silently dropped
recommendation: when a future increment wires `mcp.yml` writing into a
real scaffold/init path (or wires `homeDir` through `runDoctorChecks` for
some other reason), a `TC-014`-style check reusing
`loadMcpProjectConfig`/`validateMcpProjectConfig`'s existing
`McpConfigValidationIssue` `rule` ids (already shaped for exactly this
consumption, per the implementer's own design) should be added at that
point — not before.

### 6. `@axiom/isolation` multi-repo extension (Q-mcp-2) — re-confirmed still correctly deferred

Re-checked during this review: no new call site requiring
`buildIsolationContext` to accept a multi-repo `ProjectEntryV2` has
appeared in any of this chain's three increments' changes. The audit's
original finding (no live production code path needs this extension)
still holds. Remains an explicitly documented future consideration, not
implemented — consistent with `Axiom.SDD/AGENTS.md`'s anti-speculative-
architecture rule.

## Validation

Real validation commands were discovered and run (package scripts in
`Axiom/package.json`): `npm run typecheck` (`tsc -b`), `npm run build`
(`tsc -b`), `npm test` (`vitest run`).

- `npm run typecheck`: **clean, no errors** (whole monorepo, after the
  `registry.ts` fix).
- `npm run build`: **clean, no errors** (whole monorepo, after the fix).
- `npm test` (full monorepo, after the fix): **12 failed files / 13
  failed tests / 1513 passed** — byte-identical failing-test-name set to
  the independently-reconfirmed clean-`git HEAD` baseline (verified via
  direct diff of the two captured failure lists, zero difference).
- Scoped run (`packages/user-workspace packages/adapters/opencode
  packages/adapters/claude-code packages/installer apps/cli`): **1
  pre-existing failure** (`start.test.ts` Scenario 4, part of the
  confirmed 13-test baseline, real `profiles.yaml` `ENOENT` at
  `process.cwd()`), all other files pass, including all of this
  increment's own new tests (`mcp-config.test.ts`, both `mcp-json.test.ts`
  files) and `document-bootstrap.test.ts` (both scenarios now pass after
  the fix).

## Result

**Discrepancy investigation result: the implementer's claim was
INCORRECT.** The 13/14-vs-12/13 delta was a real, narrow regression
caused directly by this increment's `GENERATED_FILES_BY_TARGET` change
in `Axiom/packages/installer/src/registry.ts` (declaring
`.opencode/mcp.json`/`.claude/mcp.json` ahead of any actual writer),
which broke `apps/cli/src/commands/configure.ts`'s `generatedFiles`
exact-count summary (a consumer that treats the array's `.length` as a
literal produced-file count, not merely a write-scope allow-list). Fixed
by reverting those two entries with an explicit code comment explaining
the deferral, restoring `opencode`/`claude-code`'s pre-increment file
lists. Post-fix, the full-suite failure set is byte-identical to the
independently-reconfirmed clean-`HEAD` baseline (12 failed files/13
failed tests), with zero attributable regressions from this chain's
changes.

Independent review of the implementer's other four claims:

- All six `validateMcpProjectConfig` rules are **confirmed genuinely
  enforced** by direct code trace (see Implementation notes §2).
- Q-mcp-1's separation decision is **confirmed, not overridden** (see §3)
  — `mcp.ts`'s fresh reading shows no overlapping consumers and no
  user-facing confusion risk serious enough to warrant reconciliation.
- `generateOpencodeMcpJson`'s transformation logic is **confirmed
  correct** against a realistic fixture, traced directly (see §4);
  `generateClaudeCodeMcpJson` confirmed structurally identical.
- The recommended `@axiom/doctor` check category is **explicitly
  deferred** (see §5): not cheap (needs a `runDoctorChecks` signature
  change to thread `homeDir` for the registry-dependent rules) and no
  live `mcp.yml` writer exists yet to validate against.

`npm run typecheck` and `npm run build` pass clean across the whole
monorepo. `npm test`'s full-suite failure set now matches the
independently re-verified pre-existing baseline exactly.

## General spec integration

Added a new "MCP project config (`mcp.yml`)" section to
`Axiom.Spec/general-spec.md`, consolidating: the schema (`McpProjectConfig`,
`McpServerEntry`), the six validation rules and their stable `rule` ids,
the two generator functions and their output paths, the Q-mcp-1
separation decision and its rationale (explicitly distinguishing
`mcp.yml` from `mcp-manifest.yaml`), and the two explicitly-deferred
items (Q-mcp-2's `@axiom/isolation` multi-repo extension, and the
`@axiom/doctor` `mcp.yml` check category).

## INC-15 closure summary

**INC-15 ("Project-scoped MCP isolation + per-adapter MCP config") is
CLOSED as a whole.** All three roles in its chain are complete:

1. **Audit** (`INC-20260702-mcp-isolation-config-reconcile`): confirmed
   `@axiom/isolation`'s single-repo-root assumption is accurate but has
   no live consumer needing multi-repo support; confirmed registry v2
   already answers both of addendum §17's explicit validation asks;
   confirmed no persistent cache exists to namespace; confirmed
   `@axiom/mcp-tools`'s dispatch is structurally single-project-per-call;
   designed `mcp.yml`'s schema and six validation rules.
2. **Implementer** (`INC-20260702-mcp-isolation-config-reconcile-impl`):
   implemented `mcp-config.ts` (types, loader, validator) in
   `@axiom/user-workspace`; implemented per-adapter `mcp.json` generators
   for opencode and claude-code; resolved Q-mcp-1 (kept `mcp.yml` and
   `mcp-manifest.yaml` separate); wrote 21 new tests.
3. **Validator-reviewer** (this increment): caught and fixed a real
   regression the implementer's own validation had mischaracterized as
   pre-existing; independently confirmed all six validation rules, the
   Q-mcp-1 decision, and the adapter generation logic directly against
   source; explicitly deferred the doctor-check recommendation with
   documented reasoning; ran full real validation; integrated stable
   knowledge into `general-spec.md`.

**Deferred items, named explicitly** (not silently dropped):

- **Q-mcp-2**: `@axiom/isolation`'s `buildIsolationContext`/
  `pathsAreIsolated` multi-repo (`ProjectEntryV2`) extension. No live
  call site needs it; building it now would be speculative architecture
  per `Axiom.SDD/AGENTS.md`.
- **`@axiom/doctor` `mcp.yml` check category**: deferred because (a) it
  needs a `runDoctorChecks` signature change to thread `homeDir` for its
  two registry-dependent rules, and (b) no installer/scaffold path writes
  `mcp.yml` to any real project yet.
- **`mcp.yml` -> adapter generator wiring** into `runConfigure`/`sync`'s
  actual install/sync pipeline: the generator functions
  (`generateOpencodeMcpJson`/`generateClaudeCodeMcpJson`) are fully
  implemented and tested but not called from any production CLI flow —
  explicitly left for a future `cli-implementer` pass, per the
  implementer's own non-goal.
- **CLI subcommand for `mcp.yml` validation** (e.g. `axiom mcp-config
  validate`): left to a future increment if a concrete need arises; the
  loader/validator is a library API today, consumable by a future CLI
  command.
- **Generation for the remaining four adapters** (github-copilot,
  vscode, cursor, litellm): `litellm` is a proxy with
  `GENERATED_FILES_BY_TARGET: []` by design (not a candidate); the other
  three are left for a future increment if a concrete need arises.

## Next step recommendation

INC-15 closes cleanly. Per the parent roadmap
(`Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/
README.md`), the next increment in sequence is **INC-16 ("Optional
external integrations as plugins")**.
