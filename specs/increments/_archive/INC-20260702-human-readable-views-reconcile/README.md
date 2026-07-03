# Increment: Human-readable generated views (audit + value assessment)

Status: closed
Date: 2026-07-02

## Goal

Audit whether a dedicated, generated human-readable aggregate view (e.g. a
`REGISTRO_INCREMENTOS.md`-style file, addendum §7) is genuinely needed on
top of what already exists, and either implement the minimal version or
document why it is not needed yet. This is the parent roadmap's INC-12
(`Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/README.md`),
explicitly marked **optional, low priority** there and framed as optional
by addendum §7 itself (three options: manual+curated, generated+gitignored,
or generated-for-publication — none mandatory).

This increment combines the audit (normally a separate migration-engineer
pass) with the index-engineer role, per the task brief, since the roadmap
marks this increment as low-priority/optional and a full separate audit
pass would be disproportionate.

## Context

Dependencies per roadmap: INC-06 (folder-per-artifact convention, closed)
and INC-07 (`listArtifacts`/`axiom index rebuild`/`validate`, closed) —
both already provide the raw material any generated view would consume.

Addendum §7's exact framing (`axiom_decisiones_sesion_addendum_revision.md`,
section 7): aggregated human registries "pueden ser vistas generadas, no
fuente primaria." Three options: (1) manual, curated, versioned by humans;
(2) generated + gitignored, if it only replicates metadata; (3) generated
for publication/documentation. Source of truth remains `metadata.yml` per
artifact + the artifact's own content, regardless of which option (if any)
is chosen.

## Scope

- Audit whether anything resembling a generated aggregate human-readable
  view already exists in `Axiom/packages/*` or `Axiom/apps/cli/*`.
- Assess whether `listArtifacts` (INC-07) plus the existing
  `axiom-{increment,bug,plan,adr,decision} list` commands (INC-06) and
  `axiom index rebuild`/`validate` (INC-07) already give a human everything
  a generated aggregate file would provide.
- Make an explicit value judgment: build the minimal version now, or close
  as "audited, not needed yet."
- If value is found: implement the smallest possible version (one command,
  reusing `listArtifacts` directly, gitignored output, no new persistent
  state).

## Non-goals

- No curated/versioned registry (addendum §7 option 1) — no editorial
  content exists to curate; this would be speculative.
- No publication-oriented rendering pipeline (addendum §7 option 3) — no
  publication consumer exists today.
- No new persistent cache or index file beyond what INC-07 already
  provides (cache-less, direct-scan).

## Acceptance criteria

- [x] Confirmed whether `@axiom/skills`'s status reporting or
      `@axiom/doctor`'s evidence strings already function as a generated
      aggregate view (they do not — they are per-item/per-check fields,
      not an aggregate document).
- [x] Confirmed whether `listArtifacts` already provides everything needed
      to trivially generate an all-kinds Markdown view on demand (it does).
- [x] Confirmed whether `axiom-{kind} list` commands already print
      id/title/status per kind on demand (they do, including `--json`).
- [x] Confirmed whether `axiom index rebuild` already aggregates counts
      across ALL kinds (increment/bug/plan/adr/decision) in one command
      (it does).
- [x] Explicit value judgment made and documented, with a stated
      concrete-consumer test.
- [x] If judged not needed: no-code rationale documented, consistent with
      INC-05's (`INC-20260702-agents-md-adapter-reconcile`) precedent.
- [x] Recorded in `Axiom.Spec` per `Axiom.SDD/AGENTS.md`; no code changed
      in `Axiom.SDD` or `Axiom` for this increment.

## Open questions

- None blocking. If a future concrete consumer emerges (e.g. an MCP tool
  that wants a single file instead of N `listArtifacts` calls, or a
  documentation-publication pipeline), re-open as its own small increment
  with that consumer named explicitly.

## Assumptions

- "Concrete value" requires a named consumer or workflow that `list`/
  `index rebuild` cannot already serve (e.g. a CI step, a documentation
  site, an MCP tool reading a file instead of calling code). Absent that,
  building a generated file is speculative infrastructure per
  `Axiom.SDD/AGENTS.md`'s explicit bootstrap limits ("Do not introduce...
  unless explicitly requested," "do not create speculative architecture").

## Audit findings

### 1. No generated aggregate view exists anywhere today

Repo-wide search (`REGISTRO`, "generated view", "human-readable",
"Markdown summary", case-insensitive) across `Axiom/` found no file or
generator matching this pattern. The nearest per-item analogs:

- `@axiom/skills`'s catalog `status`/`securityCheckStatus` fields
  (`Axiom/packages/skills/src/catalog.ts`) — review-state metadata on a
  single catalog entry, not an aggregate document.
- `@axiom/doctor`'s `DoctorCheck { id, category, description, status,
  evidence }` shape (`Axiom/packages/doctor/src/checks.ts`) — one
  evidence string per check, printed as part of a doctor run, not a
  standalone artifact registry.

Neither is a "view" in the addendum §7 sense (a human skimming all
artifacts of a kind, or across kinds, without running a command per kind).

### 2. `listArtifacts` already makes a Markdown view trivial to generate on demand

`listArtifacts(projectRoot, kind)` (`Axiom/packages/workflow/src/
artifact-store.ts:631`) returns `{ entries: [{id, metadata}], failures:
[] }` for any `ArtifactKind`, tolerating an absent kind folder and
individual malformed `metadata.yml` files. `metadata` includes `title`
and `status` for every entry. This is already sufficient input for a
Markdown table generator — no new data-access code would be needed, only
a formatting/write step.

### 3. `axiom-{kind} list` already covers the per-kind human view, on demand

`runListCommand`/`addListSubcommand` (`Axiom/apps/cli/src/commands/
artifact-metadata-cli.ts:348-400`) already print exactly
`<id>  [<status>]  <title>` per entry for a single kind, with a `--json`
flag for machine consumption, wired into `axiom-increment`, `axiom-bug`,
`axiom-plan`, `axiom-adr`, `axiom-decision` (INC-06/INC-11). This is a
live, always-current view — never stale, because it is a direct scan, not
a cached/generated file.

### 4. `axiom index rebuild` already covers the ALL-kinds-at-a-glance case

`runIndexRebuild` (`Axiom/apps/cli/src/commands/index-cmd.ts:97-125`,
INC-07) already iterates `SCANNED_KINDS = ['increment', 'bug', 'plan',
'adr', 'decision']` in one call and prints a per-kind count + failure
count + total, in one command, with `--json` support. This is exactly the
"human skimming the repo without running N commands" use case the task
brief hypothesized as the one scenario that might justify a separate
generated file — and it is already served today by a command that exists
for an unrelated reason (index reconciliation), at no extra cost.

## Value assessment

**No genuine incremental value identified for building a generated file
right now.** The addendum's own justification for a generated view
(§7: "para publicación/documentación" or "vista generada... si sólo
replica metadata") requires either (a) a publication/documentation
consumer, or (b) a use case where a live command is insufficient (e.g.
offline reading, git-diffable history, CI artifact). None of these exist
today:

- No documentation-publication pipeline reads or would read such a file.
- No CI step is defined that would consume it.
- The addendum mandates the file be gitignored if it only replicates
  metadata (option 2) — a gitignored file provides no git history/diff
  value either, so it would not even serve as a durable record.
- The "one glance across all kinds" need is already met by `axiom index
  rebuild` (counts) and, per-kind, by `list` (full id/title/status).
  The only genuinely new thing a generated file would add is an
  **all-kinds, full-entry-detail** table (not just counts) in one
  artifact — a real but marginal convenience, not a capability gap.

This is consistent with INC-05's precedent
(`INC-20260702-agents-md-adapter-reconcile/README.md`): when audited code
already satisfies the addendum's intent without new work, the correct
action is to close with "no code needed," not to build speculative
infrastructure to fill a gap that does not exist. It is also consistent
with the roadmap's own framing of INC-12 as "optional, low priority."

## Implementation notes

Audit only. No files were changed under `Axiom/packages/`, `Axiom/apps/`,
or `Axiom.SDD/`.

### Files read in full or in relevant part

- `Axiom/packages/workflow/src/artifact-store.ts` (`listArtifacts` and
  surrounding context).
- `Axiom/apps/cli/src/commands/artifact-metadata-cli.ts`
  (`runListCommand`, `addListSubcommand`).
- `Axiom/apps/cli/src/commands/axiom-increment.ts` (list wiring).
- `Axiom/apps/cli/src/commands/index-cmd.ts` (`runIndexRebuild`,
  `runIndexValidate`, `SCANNED_KINDS`).
- `Axiom/packages/skills/src/catalog.ts` (`status`/`securityCheckStatus`
  fields).
- `Axiom/packages/doctor/src/checks.ts` (`IX-001`, `DoctorCheck` shape,
  spot-checked).
- Repo-wide case-insensitive search for `REGISTRO`, "generated view",
  "human-readable", "Markdown summary" — no matches beyond this
  increment's own new spec file.
- `Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/
  README.md` (INC-12 entry, full roadmap context).
- `axiom_decisiones_sesion_addendum_revision.md` section 7, read directly
  (not paraphrased from the roadmap summary).
- `Axiom.Spec/specs/increments/INC-20260702-agents-md-adapter-reconcile/
  README.md` (INC-05 "no work needed" precedent, read in full for
  closure-phrasing consistency).

## Validation

`No validation command was found. Performed best-effort validation by inspecting changed files and checking consistency against the requested behavior.`

No code was changed by this increment, so there is nothing to build/test/
typecheck. Best-effort validation performed: every claim above is sourced
from directly reading the named files (line numbers cited), not inferred
from package/file names, and cross-checked against a repo-wide search
confirming no existing generated-view artifact was missed.

## Result

Audited `listArtifacts`, the `axiom-{kind} list` commands (INC-06), and
`axiom index rebuild`/`validate` (INC-07), plus `@axiom/skills` status
fields and `@axiom/doctor` evidence strings as the task brief specifically
asked to check. Found no existing generated aggregate view, but also found
that the existing on-demand commands (`list` per kind, `index rebuild`
across all kinds) already cover every concrete use case addendum §7
describes, with the sole exception of an all-kinds *full-detail* table in
a single artifact — a marginal convenience with no named consumer today.
Judged this increment **not worth building yet**: no concrete
consumer/use case beyond what `list`/`index rebuild` already provide,
consistent with the roadmap's own "optional, low priority" framing and
INC-05's precedent for closing audits that find no gap.

## General spec integration

No integration into `Axiom.Spec/general-spec.md` was performed. The one
fact worth carrying forward if a future increment revisits this: **do not
propose a generated aggregate view again without first naming a concrete
consumer** (CI step, documentation pipeline, or MCP tool that specifically
needs a file rather than a `listArtifacts` call) — the existing `list`
(per-kind) and `index rebuild` (all-kinds counts) commands already cover
every use case identified in this audit. This is a minor,
narrowly-scoped judgment call rather than new stable architecture, so it
is recorded here rather than promoted into `general-spec.md`.

## Next step recommendation

**Close this increment.** No further subagent (docs-skills-writer,
validator-reviewer) has concrete work to do, because no code was written
and no gap was found. If a concrete consumer for an aggregate generated
view emerges later (e.g. a documentation-publication pipeline, or an MCP
tool that prefers reading one file over calling `listArtifacts` N times),
open a new, small, explicitly-scoped increment naming that consumer, and
implement per addendum §7 option 2 (generated + gitignored) as the
default choice unless that future consumer specifically needs option 3
(publication rendering).
