# Increment: Validate and close INC-03 narrow-scope (validator-reviewer)

Status: closed
Date: 2026-07-02

## Goal

Independently verify, without trusting any prior role's spec at face
value, the full INC-03 (narrow-scope) chain: migration-engineer's audit,
docs-skills-writer's canonical `AGENTS.md` generator, and
registry-engineer's "confirm-only, no gap" finding. Run full monorepo
validation and, if everything holds, close INC-03 as a whole with an
explicit list of deferred items. This is the `validator-reviewer` role,
the final step in INC-03's sequence (migration-engineer ->
docs-skills-writer -> registry-engineer -> **validator-reviewer**).

## Context

Parent roadmap: `Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/README.md`,
Phase B ("Instalación de proyecto"), INC-03, under an explicit
user-approved narrow scope: Roles (8-role model) and Tools/MCPs (MCP
selection + `mcp.yml`) dimensions are deferred to future increments, not
built now.

Three prior specs in this chain (all read in full before this role's
work):

1. migration-engineer audit —
   `Axiom.Spec/specs/increments/INC-20260702-installer-wizard-reconcile/README.md`
   (Status: pending). Audit-only, no code changes. Produced a 6-dimension
   gap analysis and recommended docs-skills-writer create (not
   extend/verify) a canonical `AGENTS.md`.
2. docs-skills-writer —
   `Axiom.Spec/specs/increments/INC-20260702-installer-wizard-reconcile-agents-md/README.md`
   (Status: pending). Implemented `renderCanonicalAgentsMd` +
   `writeCanonicalAgentsMd` in `@axiom/document-bootstrap`, wired
   additively into `init.ts`. Claimed baseline: 12 failed files / 13
   failed tests / 1231 passed.
3. registry-engineer —
   `Axiom.Spec/specs/increments/INC-20260702-installer-wizard-reconcile-registry-wiring/README.md`
   (Status: closed). Confirm-only: found `init.ts` already correctly
   wires `addProjectV2` per INC-01's shape; no code changed. Recommended
   skipping cli-implementer and proceeding straight to validator-reviewer.

## Scope

- Confirm the `Axiom/` working tree state matches what docs-skills-writer
  and registry-engineer claim (uncommitted diff inspection via `git
  status`/`git diff`).
- Re-read `axiom_decisiones_sesion_prompt_implementacion.md` §18.1/§18.2
  directly and byte-diff the source doc's verbatim fenced blocks against
  the two constants in `canonical-agents-md.ts`.
- Re-read `@axiom/installer`'s `registry.ts`
  (`GENERATED_FILES_BY_TARGET`) directly to independently verify the
  path-collision claim (repo-root `AGENTS.md` vs.
  `.opencode/AGENTS.md` / `.antigravity/AGENTS.md`).
- Re-read `init.ts`'s live `addProjectV2` call site and
  `registry-types.ts`'s `ProjectEntryV2`/`RegistryRepoEntry` shapes
  directly to independently verify registry-engineer's "no gap" finding.
- Run `npm run typecheck`, `npm run build`, `npx vitest run` (full
  monorepo) and compare against the stated baseline (12 failed files /
  13 failed tests / 1231 passed).
- Assess INC-03-as-a-whole's acceptance criteria (as narrow-scoped) and
  close if met, with an explicit, named list of deferred follow-ups.

## Non-goals

- No new code implementation. This is a verification-and-closure role.
- No re-opening of the Roles/Tools-MCPs scope decision (already
  explicitly deferred by direct user instruction, confirmed again here).
- No fix to the `slugifyProjectId`/`PROJECT_NAME_REGEX` edge case
  (independently confirmed real, but out of narrow scope — same
  conclusion registry-engineer reached).
- No `schemaVersion: 2` cutover (tracked separately).

## Acceptance criteria

- [x] `Axiom/` working tree state confirmed: canonical `AGENTS.md`
      generator + tests present and uncommitted; `init.ts` wiring
      present and uncommitted; registry-engineer made zero code changes.
- [x] Source doc §18.1/§18.2 re-read directly; generator output
      byte-diffed against verbatim source text — confirmed identical,
      not just structurally similar.
- [x] Path-collision claim independently verified by reading
      `GENERATED_FILES_BY_TARGET` directly (not trusting the prior
      spec's description).
- [x] Registry-engineer's "confirm-only, no gap" finding independently
      verified by reading `init.ts`'s live `addProjectV2` call and
      `registry-types.ts` directly.
- [x] `npm run typecheck`, `npm run build`, `npm test` (full monorepo)
      executed; results compared against stated baseline — confirmed
      identical, zero regressions.
- [x] INC-03-as-a-whole acceptance criteria assessed against the narrow
      scope; closed with an explicit deferred-items list.
- [x] Recorded in `Axiom.Spec` per `Axiom.SDD/AGENTS.md`.
- [x] Next-step recommendation given (INC-04).

## Open questions

None blocking. Two non-blocking observations, both already flagged by
prior roles and re-confirmed here as accurate, are carried forward as
named follow-ups (see "Deferred items" below): the `slugifyProjectId`/
`PROJECT_NAME_REGEX` edge case, and whether `configure.ts` should
backfill the canonical `AGENTS.md` for pre-existing projects.

## Assumptions

- "Independently verify" means re-reading primary sources (the external
  decision doc, the actual `Axiom/` source files, `git diff` output) and
  re-deriving each claim from them, rather than trusting the prior
  specs' prose — consistent with this role's explicit instructions.

## Implementation notes

### Files/sources read directly for independent verification

- `C:\Users\igutierrezz\Downloads\axiom_decisiones_sesion_prompt_implementacion.md`,
  §18 (lines 1539–1579), extracted verbatim.
- `Axiom/packages/document-bootstrap/src/canonical-agents-md.ts` (full
  file).
- `Axiom/packages/document-bootstrap/src/index.ts` (full file).
- `Axiom/packages/installer/src/registry.ts` (full file).
- `Axiom/apps/cli/src/commands/init.ts` (`git diff` + live
  `addProjectV2` call site, lines ~699–705).
- `Axiom/apps/cli/src/commands/repo.ts` (`runRepoAttach`'s
  `addProjectV2` call site, lines ~242–248, for comparison).
- `Axiom/packages/user-workspace/src/registry-types.ts`
  (`RegistryRepoEntry`, `ProjectEntryV2`).
- `Axiom/packages/user-workspace/src/registry-id.ts`
  (`slugifyProjectId`).
- `Axiom/apps/cli/src/commands/init.ts` (`PROJECT_NAME_REGEX`).
- `git status --porcelain=v1` / `git diff --stat` on `Axiom/`.

### Finding V1 — Working tree state matches both prior specs' claims exactly

`git status --porcelain=v1` on `Axiom/` shows exactly:

```
 M apps/cli/src/commands/init.ts
 M apps/cli/tests/init.test.ts
 M packages/document-bootstrap/src/index.ts
?? packages/document-bootstrap/src/canonical-agents-md.ts
?? packages/document-bootstrap/tests/canonical-agents-md.test.ts
```

`git diff --stat` confirms 3 changed files, 88 insertions, 0 deletions —
no unrelated files touched. This is exactly the "Files changed" set
docs-skills-writer's spec declares, and confirms registry-engineer's
claim of zero code changes: there is nothing in the tree beyond
docs-skills-writer's diff. No drift found.

### Finding V2 — Canonical `AGENTS.md` generator matches source doc §18.1/§18.2 word-for-word (programmatically verified, not eyeballed)

Extracted the two fenced `md` blocks from
`axiom_decisiones_sesion_prompt_implementacion.md` §18.1 ("Regla
estructural común") and §18.2 ("Regla de separación de concerns")
directly from the source file, and extracted the two template-literal
constants (`STRUCTURAL_INTEGRITY_RULES_BLOCK`,
`SEPARATION_OF_CONCERNS_BLOCK`) directly from
`canonical-agents-md.ts` via a small Node script (string equality
check on trimmed content, not visual inspection). Result:

```
=== BLOCK 1 (structural integrity) ===
source doc === impl (trimmed): true
=== BLOCK 2 (separation of concerns) ===
source doc === impl (trimmed): true
```

Both blocks are byte-identical to the source doc, including exact
line breaks, bullet formatting, and the closing `axiom doctor` line (no
trailing period added, no rewording of "Never edit generated
index/cache files manually," no paraphrasing of the CLI command list).
**No drift found.** The docs-skills-writer spec's claim of "verbatim"
is accurate, not just structurally similar.

### Finding V3 — Path-collision claim independently confirmed from `GENERATED_FILES_BY_TARGET` directly

Read `Axiom/packages/installer/src/registry.ts` in full.
`GENERATED_FILES_BY_TARGET` maps exactly 8 adapter target ids to file
lists:

- `opencode` -> `['.opencode/AGENTS.md', '.opencode/skills-lock.yaml']`
- `copilot-vscode` -> `['.vscode/copilot-instructions.md', '.vscode/settings.json']`
- `claude-code` -> `['.claude/CLAUDE.md', '.claude/skills-lock.yaml']`
- `antigravity` -> `['.antigravity/AGENTS.md']`
- `visual-studio-2026` -> `['.vs/AXIOM.md']`
- `cursor` -> `['.cursor/rules/axiom.mdc']`
- `github-copilot` -> `['.github/copilot-instructions.md']`
- `litellm` -> `[]`

No entry in this map resolves to a bare `AGENTS.md` at the repo root —
every path is nested under an adapter-specific subdirectory
(`.opencode/`, `.antigravity/`, etc.). `writeCanonicalAgentsMd`'s
default target (`DEFAULT_TARGET_REL = 'AGENTS.md'`, resolved against
`projectRoot`) is a distinct, unnested path. **No path collision exists
between the canonical root `AGENTS.md` and any adapter-scoped output.**
This independently confirms the docs-skills-writer spec's claim,
verified from the registry map itself rather than from that spec's
prose description.

### Finding V4 — Registry-engineer's "confirm-only, no gap" finding independently confirmed from live `init.ts`

Read `init.ts`'s live `addProjectV2` call site directly (not via `git
diff`, but the current file content at lines ~699–705):

```ts
const addResult = addProjectV2(homeDir, {
  id: slugifyProjectId(result.projectName),
  name: result.projectName,
  repos: {
    [DEFAULT_REPO_ROLE]: { role: DEFAULT_REPO_ROLE, path: resolvedRoot },
  },
});
```

Cross-checked against `registry-types.ts`'s `ProjectEntryV2` (`repos:
Readonly<Record<RepoRoleKey, RegistryRepoEntry>>`) and
`RegistryRepoEntry` (`{ role: RepoRoleKey; path: string }`): the call
site satisfies the contract exactly — `id` is a real slug (not a
duplicated `name`), `repos` is a `{ roleKey: { role, path } }` map with
the map key and the `role` field kept consistent
(`DEFAULT_REPO_ROLE` = `'sdd'` in both places).

Cross-checked against `repo.ts`'s `runRepoAttach` call site (lines
~242–248):

```ts
const addResult = addProjectV2(ensured.homeDir, {
  id: args.repoId,
  name: resolution.name,
  repos: {
    [role]: { role, path: resolution.rootPath },
  },
});
```

Structurally identical pattern (map-keyed-by-role, `{role, path}` per
entry), confirming registry-engineer's claim that `init.ts` and
`repo.ts` are consistent with each other. **No gap found; registry-
engineer's "confirm, not implement" conclusion is independently
verified accurate**, not merely trusted from that spec's prose.

Also independently re-verified the `slugifyProjectId` /
`PROJECT_NAME_REGEX` edge case registry-engineer flagged as
non-blocking: `PROJECT_NAME_REGEX = /^[a-z0-9][a-z0-9-]{0,62}$/` accepts
consecutive hyphens; `slugifyProjectId` collapses `-+` to a single `-`
(`registry-id.ts` line 46). The claimed divergence for inputs like
`my--project` is real and correctly characterized as pre-existing,
narrow, and out of this chain's scope to fix.

### Finding V5 — `init.ts` wiring for the canonical `AGENTS.md` call is additive/best-effort as claimed

`git diff apps/cli/src/commands/init.ts` shows the
`writeCanonicalAgentsMd` call wrapped in `try/catch`, placed
immediately after `axiom.yaml` is written (before the
`installed-multi-repo` topology branch), appending to `filesCreated` on
success and writing a `[axiom init] WARN: ...` line to stderr on any
failure (`Result.err` or thrown exception) without re-throwing.
Confirms the docs-skills-writer spec's "additive, non-blocking" claim
by direct diff inspection, not by trusting its prose.

## Validation

Validation discovery order followed `Axiom.SDD/AGENTS.md` (README ->
package scripts -> task runners -> test configs -> build configs ->
available scripts). The monorepo has `npm run build`, `npm run
typecheck`, and `npx vitest run` at the root — all three were run.

Commands executed and results:

- `npm run typecheck` (full monorepo, `tsc -b`) — clean, no errors.
- `npm run build` (full monorepo, `tsc -b`) — clean, no errors.
- `npx vitest run` (full monorepo) — **12 failed files / 13 failed
  tests / 1231 passed** (125 files, 1244 tests total).

### Baseline comparison

Stated baseline (from docs-skills-writer's and registry-engineer's
specs, both already confirmed clean through the INC-01+INC-02+INC-03
chain): 12 failed files / 13 failed tests / 1231 passed.

This run: 12 failed files / 13 failed tests / 1231 passed — **identical,
byte-for-byte**. No code was changed by this role, so this is the
expected outcome (not merely "no regressions").

The list of 13 failing tests across 12 files was independently
enumerated and cross-checked: `apps/cli/tests/start.test.ts`,
`packages/agents/tests/catalog.test.ts`,
`packages/doctor/tests/checks.test.ts`,
`packages/model-routing/tests/{assignments,loader,opencode-projection,resolver}.test.ts`,
`packages/project-resolution/tests/resolver.test.ts`,
`packages/skills/tests/catalog.test.ts`,
`packages/telemetry/tests/audit-trail-sink.test.ts`,
`packages/toolchain/tests/repair-add-gitignore.test.ts`,
`packages/tui/tests/driver.test.ts` (2 tests). All are
environment/real-repo-relative-path-dependent, pre-existing failures —
none touch `document-bootstrap`, `init.ts`, `init.test.ts`, or any file
changed in this INC-03 chain. **Zero regressions, independently
confirmed.**

## Result

All four independent verification tasks confirmed the prior three
specs' claims accurate, with no drift found:

1. Working tree state matches exactly what docs-skills-writer claims
   changed and what registry-engineer claims it left untouched.
2. The canonical `AGENTS.md` generator's two boilerplate blocks are
   byte-identical to source doc §18.1/§18.2 (programmatically verified),
   not paraphrased or reformatted.
3. The path-collision claim holds: no entry in `installer/registry.ts`'s
   `GENERATED_FILES_BY_TARGET` produces a root-level `AGENTS.md`.
4. `init.ts`'s live `addProjectV2` call matches the
   `{ roleKey: { role, path } }` shape and is structurally consistent
   with `repo.ts`'s call site — registry-engineer's "no gap" finding
   holds under independent re-reading.

Full monorepo `typecheck`/`build`/`test` are clean/identical to the
stated baseline (12 failed files / 13 failed tests / 1231 passed) —
zero regressions.

## INC-03 (narrow-scope) closure assessment

INC-03's own acceptance criteria, as explicitly narrowed by direct user
instruction (Roles and Tools/MCPs deferred, not built now), are fully
met:

- **Audit done**: migration-engineer's 6-dimension gap analysis, fully
  evidence-grounded (file/function citations), closed the "understand
  current state vs. target" step.
- **Canonical `AGENTS.md` created**: `renderCanonicalAgentsMd` +
  `writeCanonicalAgentsMd` implemented, wired additively into `init.ts`,
  verbatim §18.1/§18.2 content confirmed byte-identical, no path
  collision with adapter-scoped outputs, 11 new passing tests, zero
  regressions.
- **Registry wiring confirmed**: `init.ts`'s `addProjectV2` call
  independently re-verified to already implement the reconciled
  `{ roleKey: { role, path } }` shape from INC-01 — no gap, no code
  needed.
- **Roles/Tools-MCPs explicitly and visibly deferred, not silently
  dropped**: both dimensions are named, in writing, in three separate
  specs in this chain (migration-engineer's gap table calling them
  "large gap"; registry-engineer's explicit non-goals section;
  restated here) — not omitted or forgotten.

**INC-03 (narrow-scope) is closed as a whole.**

### Deferred items (named, tracked follow-ups — not lost work)

1. **Roles model** (8-role model: PO/Analyst/Architect/Backend/Frontend/
   QA/DevOps/Orchestrator, with per-role repo associations). Currently
   served by an unrelated concept (`functionalProfile`:
   `product-owner`/`builder`, a capability/token-budget axis, not a
   role-to-repo model). No fields were added to `ProjectEntryV2`,
   `RegistryRepoEntry`, or `axiom.yaml` for this. Needs its own future
   increment.
2. **Tools/MCPs selection + `mcp.yml`** (user-facing MCP selection: Axiom
   project MCP, Serena per repo, Context7, Azure DevOps/Jira, etc.).
   Currently `EXTERNAL_DEPS_BY_CAPABILITY` derives MCP dependencies as a
   side-effect of capability selection, not an explicit user question;
   no `mcp.yml` manifest exists. Needs its own future increment
   (parent roadmap's INC-13/15 territory per migration-engineer's audit).
3. **`schemaVersion: 2` re-enablement in `init.ts`**. Migration-engineer's
   Finding 5 established the blocking comment in `init.ts` is factually
   stale (`resolveProject` already has a working `resolveFromV2` path,
   confirmed via same-commit `git log` cross-check); re-enabling v2
   emission is empirically unblocked but deliberately scoped as a
   separate, small, independently-revertable follow-up (e.g.
   `INC-20260702-init-schemaversion2-cutover`), not folded into INC-03.
4. **`slugifyProjectId` vs. `PROJECT_NAME_REGEX` hyphen-collision edge
   case**. `PROJECT_NAME_REGEX` accepts consecutive hyphens (e.g.
   `my--project`); `slugifyProjectId` collapses them to one
   (`my-project`), so the registry's derived `id` can diverge from
   `axiom.yaml`'s `project.name` by exactly this transformation in that
   edge case. Independently re-confirmed real in this role (Finding V4).
   Pre-existing since INC-01's schema-writer design; fixing it touches
   either a validation-rule change (`PROJECT_NAME_REGEX`, blast radius
   across every `projectName` caller) or a `@axiom/user-workspace`
   package change (`slugifyProjectId`, affects `repo attach` and `join`
   too) — both exceed any single narrow-scope increment in this chain.
   Recommend a small, dedicated follow-up if it ever becomes an
   operator-facing confusion point.
5. **`configure.ts` backfill of canonical `AGENTS.md`** for projects
   `init`-ed before this increment landed. Flagged as an open question
   by docs-skills-writer, not decided in this chain — `configure.ts` has
   no natural "first write" moment the way `init` does. Left for a
   future increment once real usage surfaces the gap.
6. **Adapter reconciliation** (making `.opencode/AGENTS.md` /
   `.antigravity/AGENTS.md` derive from the new canonical `AGENTS.md`
   rather than being generated independently). Explicitly out of scope
   per docs-skills-writer's non-goals; the parent roadmap's own INC-05
   already scopes this ("AGENTS.md as canonical contract + adapter
   reconciliation").

## General spec integration

No integration into `Axiom.Spec/general-spec.md` was performed — this
file does not exist in this repo (confirmed absent, consistent with
every prior increment in this chain's own note). The one piece of
stable knowledge worth flagging for whoever eventually creates
`general-spec.md`: **INC-03 (narrow-scope) is closed** — the canonical
`AGENTS.md` contract (source doc §3/§18) is a real, implemented,
verified artifact in `Axiom/` (`@axiom/document-bootstrap`'s
`canonical-agents-md.ts`), and `axiom init`'s registry-wiring to
`addProjectV2` is a closed, verified integration point. Future work on
Roles, Tools/MCPs, `schemaVersion: 2`, the hyphen-collision edge case,
the `configure.ts` backfill, and adapter reconciliation should reference
this closure and the six deferred items above rather than re-auditing
this ground from scratch.

## Next step recommendation

**Proceed with INC-04 ("Reconcile contextual TUI shell and detection")**
per the parent roadmap
(`Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/README.md`),
now that INC-03 (narrow-scope) is closed.

Separately, the six deferred items listed above remain available to be
picked up as their own future increments whenever prioritized — none of
them block INC-04, and none of them were dropped: they are documented
here, and in migration-engineer's and registry-engineer's specs, as
named, scoped follow-ups.
