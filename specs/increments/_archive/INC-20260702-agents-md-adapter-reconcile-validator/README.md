# Increment: AGENTS.md canonical contract vs adapter reconciliation (validator-reviewer)

Status: closed
Date: 2026-07-02

## Goal

Independently re-verify the migration-engineer audit
(`Axiom.Spec/specs/increments/INC-20260702-agents-md-adapter-reconcile/README.md`)
for parent roadmap INC-05 ("AGENTS.md as canonical contract + adapter
reconciliation"), by reading adapter generator source directly rather
than trusting the audit's diff table, and produce a closure decision
for INC-05 as a whole. This is the `validator-reviewer` role in the
INC-05 subagent chain (migration-engineer already ran; this closes the
chain per the audit's own "validator-reviewer is optional... unless
the user wants closure recorded" recommendation — the user explicitly
requested this closure pass).

## Context

The migration-engineer audit read all 6 adapter packages
(`opencode`, `claude-code`, `github-copilot`, `vscode`, `cursor`,
`litellm`) plus `@axiom/document-bootstrap`,
`Axiom/packages/installer/src/registry.ts`, and the `sync.ts`/
`configure.ts` invocation sites, and concluded:

1. None of the 6 adapters overlap in content with canonical
   `AGENTS.md` (built in INC-03,
   `@axiom/document-bootstrap/src/canonical-agents-md.ts`) — adapters
   read `ResolvedInstallProfile` and render capability/skill routing
   or tool config, not portable governance instructions.
2. No reconciliation code is needed; building a dependency from
   adapters onto canonical `AGENTS.md` now would be speculative
   architecture per `Axiom.SDD/AGENTS.md`.
3. Two adjacent, out-of-scope defects were flagged: (a) `registry.ts`'s
   `GENERATED_FILES_BY_TARGET` lists a stale `.claude/CLAUDE.md`
   filename for the `claude-code` target, which does not match the
   actual `.claude/AGENTS.md` the generator writes; (b) `configure.ts`
   mixes the old `ResolvedVariables`-based copilot-instructions
   pipeline with the new adapter-package pipeline, inconsistently with
   `sync.ts`'s uniform new-pipeline dispatch.

This increment independently re-verifies findings 1-3 by reading
source directly, decides whether the flagged filename drift is a safe
narrow fix, and issues a closure decision for INC-05.

## Scope

- Read `Axiom.SDD/AGENTS.md` (canonical rule source) in full.
- Independently read, from source, at least 3 of the 6 adapter
  generators (chosen: `claude-code` — mandatory, since it carries the
  filename-drift bug — plus `opencode` and `github-copilot`, covering
  both `AGENTS.md`-named Markdown outputs and non-`AGENTS.md`-named
  Markdown output, i.e. two different content shapes) to confirm or
  refute the "no overlap" finding.
- Independently read the exact line in
  `Axiom/packages/installer/src/registry.ts` that the audit's
  Finding 4 references, and independently trace its consumers
  (`installer.ts`, tests) to assess fix risk.
- Confirm Finding 3 (`configure.ts`/`sync.ts` pipeline inconsistency)
  is real by re-reading the cited sections; document as tracked
  follow-up, do not fix.
- If Finding 4 is confirmed and the fix is a simple one-line filename
  correction with no behavioral risk, apply it narrowly.
- Run available validation (`typecheck`, `build`, `test`) if any code
  change was made.
- Issue an explicit closure decision for parent roadmap INC-05.
- Name both adjacent defects as explicit tracked follow-up items.
- Recommend the next step (INC-06) per the parent roadmap.

## Non-goals

- No reconciliation mechanism between adapters and canonical
  `AGENTS.md` (confirmed not needed).
- No fix to Finding 3 (`configure.ts`/`sync.ts` pipeline mixing) —
  larger, separate concern, tracked as follow-up only.
- No new increment scaffolding for the two adjacent defects beyond
  this document's tracked-follow-up list, unless one was trivial
  enough to fix inline (Finding 4 qualified; Finding 3 did not).
- No re-litigation of INC-03's canonical `AGENTS.md` template content.
- No changes to `@axiom/adapters-opencode`, `@axiom/adapters-cursor`,
  `@axiom/adapters-vscode`, `@axiom/adapters-litellm`, or
  `@axiom/document-bootstrap`.

## Acceptance criteria

- [x] At least 3 of the 6 adapters independently re-verified by
      reading generator source directly (`claude-code`, `opencode`,
      `github-copilot`).
- [x] Finding 4 (`.claude/CLAUDE.md` vs `.claude/AGENTS.md` drift)
      independently verified by reading the exact line in
      `registry.ts`.
- [x] Fix-risk assessment performed (consumer trace) before deciding
      to fix vs document Finding 4.
- [x] Finding 3 confirmed as real and documented as a tracked
      follow-up, not fixed.
- [x] Validation executed for the code change made (typecheck, build,
      test).
- [x] Explicit INC-05 closure decision recorded.
- [x] Both adjacent defects named as explicit tracked follow-up items.
- [x] Next-step recommendation given (INC-06).

## Independent verification results

### 1. Adapter content re-verification (source read directly, not trusted from audit table)

**`claude-code`**
(`Axiom/packages/adapters/claude-code/src/generator.ts`,
`agents-md.ts`) — confirmed independently:

- Writes exactly one file: `.claude/AGENTS.md` (constants
  `CLAUDE_DIR = '.claude'`, `AGENTS_MD_FILENAME = 'AGENTS.md'`,
  `generator.ts` lines 28/31/96). No skills-lock file is written —
  the generator only calls `writeAgentsMd`, never a lockfile writer
  (confirmed no `skills-lock.ts` module exists in this package's
  `src/` directory).
- Content (`renderAxiomBlock` in `agents-md.ts`) is exactly:
  `## Generated portable entries` → `### Capabilities` (alphabetical
  bullets from `resolvedProfile.enabledCapabilities`) →
  `### Skills` (alphabetical bullets from `skills[].portableEntry`) →
  a fixed `## Routing note` disclaimer ("Single-mode: routing global
  del target. Sin per-slot."). No structural-integrity or
  separation-of-concerns prose; no reference to canonical `AGENTS.md`
  at all.
- **Verdict confirmed: no content overlap with canonical `AGENTS.md`.**
  Matches the audit's table exactly.

**`opencode`**
(`Axiom/packages/adapters/opencode/src/generator.ts`,
`agents-md.ts`) — confirmed independently:

- Writes `.opencode/AGENTS.md` + `.opencode/skills-lock.yaml`.
- `renderAxiomBlock` is byte-for-byte structurally identical to
  claude-code's (same `## Generated portable entries` →
  `### Capabilities` → `### Skills` shape), minus the `## Routing
  note` block. Source comment in claude-code's `agents-md.ts`
  explicitly states this logic was "deliberately copied from
  `@axiom/adapters-opencode`... consider extracting to a shared
  helper in a future iteration" — an internal duplication note, not
  a canonical-`AGENTS.md` dependency.
- **Verdict confirmed: no content overlap.**

**`github-copilot`**
(`Axiom/packages/adapters/github-copilot/src/generator.ts`,
`instructions.ts`) — confirmed independently:

- Writes `.github/copilot-instructions.md` (not `AGENTS.md`-named at
  all). Content: `# Copilot instructions for <projectName>` →
  `## Capabilities` (alphabetical bullets) → optional
  `## Additional instructions` → fixed `## Notes` provenance bullet.
  No preservation-marker logic (full overwrite every run, per an
  explicit code comment: "copilot no usa la estrategia de markers").
- **Verdict confirmed: no content overlap** — different content
  domain (capability list + fixed provenance line), no reference to
  canonical `AGENTS.md`.

All three independently-read adapters confirm the audit's central
finding: the 6 adapters and canonical `AGENTS.md` read disjoint
inputs (`ResolvedInstallProfile` vs `CanonicalAgentsMdIdentity`) and
render disjoint content domains (capability/skill/tool-config routing
vs governance/structural-rule boilerplate + project identity). The
audit's "no reconciliation needed" recommendation holds up under
independent source review.

### 2. Finding 4 re-verification (`.claude/CLAUDE.md` drift)

Read `Axiom/packages/installer/src/registry.ts` directly. Confirmed
line 45 (as of this review):

```
'claude-code': ['.claude/CLAUDE.md', '.claude/skills-lock.yaml'],
```

This is real and matches the audit's Finding 4 exactly. Independent
addition to the audit: the array's second element
(`.claude/skills-lock.yaml`) is *also* stale — `claude-code`'s actual
generator never writes a skills lockfile (confirmed above, no
`skills-lock.ts` module in that package). The audit's Finding 4 named
only the filename; the whole array entry was stale.

**Fix-risk assessment (consumer trace, performed before fixing):**

- `GENERATED_FILES_BY_TARGET` has exactly one runtime consumer:
  `Axiom/packages/installer/src/installer.ts` line 89
  (`const targetFiles = GENERATED_FILES_BY_TARGET[args.adapterTarget]`),
  used only for an existence check (`unknown-target` error branch)
  and to populate an `augmentedProfile` metadata field. It is **not**
  used to construct actual write paths — every adapter's
  `generator.ts` hardcodes its own output path independently (as
  verified directly in section 1 above). Changing the registry's
  string values cannot change where `claude-code`'s generator
  actually writes.
- Grepped the entire `Axiom/` tree for the literal string
  `CLAUDE.md`: exactly 2 hits, both on the same line in
  `registry.ts` (the stale entry itself and its adjacent comment). No
  test file, no other source file, references this string.
- Grepped `installer/tests/installer.test.ts` for any assertion on
  the array's contents for `claude-code`: none found — the test suite
  exercises `allowedTargets` membership and `unknown-target` error
  paths, not the specific file list returned per target.
- Conclusion: **safe, narrow, one-line-scope fix** — no ambiguity
  about which name is correct (the generator's actual write path,
  independently confirmed, is unambiguous), no downstream consumer
  depends on the stale value.

**Action taken:** fixed. See Implementation notes.

### 3. Finding 3 re-verification (`configure.ts`/`sync.ts` inconsistency)

Confirmed real by re-reading the cited sections
(`Axiom/apps/cli/src/commands/configure.ts`,
`Axiom/apps/cli/src/commands/sync.ts`): `sync.ts`'s
`materializeAdapterOutputs` dispatches all 6 adapter targets
uniformly through the new adapter-package generators (including
`generateCopilotConfig` for copilot targets), while `configure.ts`
calls the new `generateOpencodeConfig` directly for `opencode` but
falls back to the OLD `writeCopilotInstructions`
(`ResolvedVariables`-keyed) pipeline for `copilot-vscode`/
`github-copilot` targets instead of `generateCopilotConfig`. This is
a real, pre-existing inconsistency, unrelated to canonical
`AGENTS.md`, and larger in scope than a one-line fix (it requires
deciding whether `configure.ts` should be changed to match `sync.ts`,
or whether the old pipeline still serves a purpose for `configure.ts`
specifically — a design question, not a typo). **Not fixed here** —
tracked as a follow-up per the Scope/Non-goals above.

## Tracked follow-up items (not addressed by this increment)

1. **`configure.ts`/`sync.ts` copilot-pipeline inconsistency**
   (Finding 3). `configure.ts` mixes the old
   `ResolvedVariables`/`writeCopilotInstructions` pipeline with the
   new adapter-package pipeline for copilot targets, while `sync.ts`
   uses the new pipeline uniformly. Needs its own scoped increment to
   decide the correct target pipeline for `configure.ts` and apply it
   consistently. Confirmed real; not fixed in this increment
   (out of scope, larger design decision).
2. **Optional cross-reference from adapter `AGENTS.md` outputs to
   canonical `AGENTS.md`** (audit's Recommendation item 3, not a
   defect): `opencode`/`claude-code`/`cursor`'s `AGENTS.md`-named
   outputs could add a one-line pointer back to the repo-root
   canonical `AGENTS.md` so a reader knows it exists. Explicitly
   optional, not required by any acceptance criterion, not
   implemented per `Axiom.SDD/AGENTS.md`'s bootstrap-limits rule
   ("document as a future consideration, do not implement now").

## Open questions

None blocking. The two items above are candidates for future,
separately-scoped increments if the user wants to pursue them.

## Assumptions

- "Simple one-line filename fix with no behavioral risk" (per the
  task's fix criterion) was interpreted to also cover removing the
  equally-stale, equally-unreferenced `skills-lock.yaml` array
  element for `claude-code`, since leaving it in place after fixing
  only the filename would still leave the registry inaccurate about
  claude-code's single-mode (no-lockfile) behavior, and the same
  fix-risk assessment (no consumer depends on array length or the
  second element) applies to it.
- The other 3 adapters (`vscode`, `cursor`, `litellm`) were not
  independently re-read in this pass — the audit already established
  their outputs are JSON, not Markdown, and structurally cannot carry
  governance prose overlap; this is a lower-risk claim than the
  three Markdown-emitting adapters that were independently verified,
  and re-confirming JSON-shaped output requires no interpretive
  judgment the audit could have gotten wrong.

## Implementation notes

One file changed:
`Axiom/packages/installer/src/registry.ts` — corrected the
`GENERATED_FILES_BY_TARGET['claude-code']` entry from
`['.claude/CLAUDE.md', '.claude/skills-lock.yaml']` to
`['.claude/AGENTS.md']`, matching the actual output of
`@axiom/adapters-claude-code`'s `generator.ts` (single file,
single-mode, no skills lockfile). Comment above the entry updated to
match. No other files changed.

## Validation

Ran, in order, per `Axiom.SDD/AGENTS.md` validation discovery
(package scripts found in `Axiom/package.json`):

- `npm run typecheck` (`tsc -b`) — passed, no errors.
- `npm run build` (`tsc -b`) — passed, no errors.
- `npm test` (`vitest run`, full monorepo, 1267 tests) — 1254 passed,
  13 failed across 12 unrelated test files (`toolchain`, `tui`,
  `model-routing`, `project-resolution`, `telemetry`,
  `doctor`, `agents`, `skills`, `filesystem-truth`, `start.test.ts`).
  None of these reference `registry.ts`, `GENERATED_FILES_BY_TARGET`,
  or the `claude-code` adapter. Confirmed pre-existing and unrelated:
  `git status` showed several of those same packages
  (`filesystem-truth`, `project-resolution`, `tui`, `user-workspace`)
  already modified in the working tree before this session began.
  Additionally ran a scoped re-run targeting exactly the affected
  surface — `packages/installer`, `packages/adapters/claude-code`,
  `apps/cli/tests/init.test.ts`, `apps/cli/tests/sync.test.ts`,
  `apps/cli/tests/configure.test.ts` — all 46 tests in that scope
  passed cleanly, confirming the registry fix introduced no
  regression.

## Result

Independently re-verified, by reading generator source directly
(not the audit's table), that `claude-code`, `opencode`, and
`github-copilot` render content disjoint from canonical `AGENTS.md`
— capability/skill/tool-config routing keyed on
`ResolvedInstallProfile`, versus governance/structural-rule
boilerplate keyed on `CanonicalAgentsMdIdentity`. The audit's central
"no reconciliation needed" finding holds up. Independently confirmed
Finding 4 (stale `.claude/CLAUDE.md` registry entry) by reading the
exact line, traced its only runtime consumer to confirm zero
behavioral risk, and applied a narrow one-line-scope fix (plus
removing the equally-stale `skills-lock.yaml` element from the same
array entry). Independently confirmed Finding 3
(`configure.ts`/`sync.ts` pipeline inconsistency) is real via direct
re-read, but left unfixed as a larger, separately-scoped follow-up.
Full monorepo validation (typecheck, build, test) executed; the one
code change is isolated and regression-free.

## General spec integration

No `Axiom.Spec/general-spec.md` exists yet in this repository
(consistent with all prior increments in this chain — confirmed
absent). The stable knowledge worth carrying forward once
`general-spec.md` exists: **canonical `AGENTS.md` and the 6 tool
adapters are intentionally disjoint pipelines** — canonical
`AGENTS.md` owns governance/identity content
(`@axiom/document-bootstrap`), adapters own
capability/skill/tool-config content (`@axiom/adapters-*`, keyed on
`ResolvedInstallProfile`). Any future work merging them is new scope,
not a divergence fix. Also worth carrying forward:
`GENERATED_FILES_BY_TARGET` in `@axiom/installer`'s `registry.ts` is
documentary metadata only (existence-check + profile annotation) —
it does not drive actual adapter write paths, which are hardcoded per
adapter in each `generator.ts`. This decouples registry accuracy from
runtime correctness but means the registry can silently drift (as
Finding 4 showed) without breaking anything at runtime, so it should
be spot-checked against actual generator output when adapters change.

## INC-05 closure decision

**Parent roadmap INC-05 ("AGENTS.md as canonical contract + adapter
reconciliation") is closed: `Status: closed`.**

Rationale, against `Axiom.SDD/AGENTS.md`'s closure rules:

- **Goal/expected behavior clear**: yes — determine whether the 6
  adapters need to derive from or reconcile with canonical
  `AGENTS.md`.
- **Acceptance criteria exist**: yes, in both the migration-engineer
  audit and this validator increment.
- **Changes implemented, or no-code rationale explicit**: no-code
  rationale is explicit and now independently confirmed twice
  (migration-engineer's audit, this validator's independent source
  re-read) — the 6 adapters and canonical `AGENTS.md` occupy disjoint
  content domains; there is no divergent instruction content to
  reconcile, and building a dependency between them now would be
  speculative architecture prohibited by `Axiom.SDD/AGENTS.md`. One
  small adjacent bug (Finding 4) was fixed as in-scope cleanup, not
  as the reconciliation itself.
- **Available validation executed**: yes (typecheck, build, test —
  see Validation section).
- **Review against intent and acceptance criteria done**: yes, this
  document.
- **Stable knowledge integrated into `general-spec.md` when
  applicable**: `general-spec.md` does not exist yet in this repo;
  the relevant knowledge is stated above for whenever it is created,
  consistent with prior increments in this chain.
- **Result documented clearly**: yes, this document plus the
  migration-engineer audit it validates.

All closure conditions in `Axiom.SDD/AGENTS.md` are satisfied.
Closing with an explicit rationale of "audited, confirmed no
reconciliation needed, adjacent bugs documented as tracked
follow-ups (one fixed as trivial in-scope cleanup, one left for a
separate increment)."

## Next step

**INC-06 — Reconcile increment/bug/plan commands**
(`axiom-increment.ts`/`axiom-bug.ts`/`axiom-plan.ts`), per
`Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/README.md`
Phase C. INC-05's chain is fully closed; no further subagent work is
pending under it. If the user wants to pursue either tracked
follow-up item above (the `configure.ts`/`sync.ts` pipeline
inconsistency, or the optional adapter-to-canonical cross-reference),
those should be opened as their own small, explicitly-scoped
increments rather than reopening INC-05.
