# Increment: Reconcile derived caches / index rebuild (migration-engineer audit)

Status: pending
Date: 2026-07-02

## Goal

Execute the **migration-engineer** step of INC-07 of the parent roadmap
(`Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/README.md`,
Phase C): audit whether `Axiom` already has any `axiom index rebuild`-equivalent
concept (derived cache, aggregate index, or a direct-scan `list` command for
increments/bugs/plans) before assuming this is additive greenfield work, per
the roadmap's own instruction ("audit whether any package already does cache
rebuild; report back before assuming additive"). This document is audit-only:
no code was changed in `Axiom`, `Axiom.SDD`, or elsewhere.

## Context

Parent chain read in full before this audit:

1. Parent roadmap —
   `Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/README.md`
   (INC-07 entry, Phase C).
2. INC-06 cli-implementer —
   `Axiom.Spec/specs/increments/INC-20260702-increment-bug-plan-commands-reconcile-impl/README.md`.
3. INC-06 validator-reviewer (closure) —
   `Axiom.Spec/specs/increments/INC-20260702-increment-bug-plan-commands-reconcile-validator/README.md`.

External decision documents re-read directly for this audit (not from
memory/summary):

- `axiom_decisiones_sesion_prompt_implementacion.md` §5 (índices: derived vs
  curated) — §5.1-5.4 specifically: `increments.index`/`bugs.index`/
  `plans.index`/`state.index`/`search.index`/`graph.index` must be a
  gitignored local cache under `.axiom/cache/`, regenerable via
  `axiom index rebuild`; source of truth stays the per-instance
  `metadata.yml` folders. §5.5: curated/versioned indexes
  (`technical-context.index`, `skills.index`, `commands.index`,
  `rules.index`, `roles.index`) are a different, out-of-scope-here concept
  ("si decide comportamiento del agente, se versiona").
- `axiom_decisiones_sesion_addendum_revision.md` §5 (rebuild automático/
  fallback) — exact rule: *"Un clone limpio del repo spec debe poder quedar
  operativo ejecutando `axiom index rebuild`."* Behavior sequence mandated:
  (1) TUI/CLI checks local cache on open, (2) if absent, offer/run
  `axiom index rebuild`, (3) if stale, regenerate, (4) if regeneration fails,
  degrade to direct scan of `metadata.yml`, (5) `axiom doctor` validates
  coherence.

**Landed state from INC-06** (confirmed by direct re-read of the code, not
trusted from the impl spec's prose): `Axiom/packages/workflow/src/
artifact-store.ts` implements a folder-per-instance `metadata.yml` store —
`<specPath>/{increments,bugs,plans}/<ID>/{README.md,metadata.yml}` — with
`generateArtifactId`/`generateUniqueArtifactId`, `loadArtifactMetadata`,
`saveArtifactMetadata` (atomic tmp+rename), `ensureArtifactReadme`, and
initial-metadata factories. `<specPath>` resolves to `<projectRoot>/
axiom.spec` (hardcoded `DEFAULT_SPEC_REL_PATH`, matching every other
CLI file's convention).

## Scope

- Grep the entire `Axiom` monorepo (`packages/*`, `apps/*`) for any existing
  index/cache-rebuild concept, by package name, exported function name, and
  CLI command/subcommand name.
- Determine whether a direct-scan `list` command for increments/bugs/plans
  exists today (would change what "index rebuild" needs to provide — value
  add vs. enabling listing at all).
- Read `@axiom/doctor/src/checks.ts` in full for its exact read/skip/fail/warn
  pattern and confirm no existing check category already does cache/index
  validation-or-rebuild-adjacent work.
- Determine, concretely, what "index rebuild" should produce given INC-06's
  landed folder-per-artifact reality.
- Check whether `~/.axiom/registry.json` (`@axiom/user-workspace`) or any
  other INC-01–05 artifact already has an analogous "regenerate from a clean
  clone" concept to pattern-match against.
- Produce a recommendation (build minimal `axiom index rebuild`, or defer as
  not-yet-needed) grounded in what was found, plus a brief for the next role
  if work proceeds.

## Non-goals

- No code implementation in this increment (audit-only, migration-engineer
  step).
- No design of the exact `.axiom/cache/*.index.json` file schema (that is
  index-engineer's job if/when this proceeds).
- No implementation of `search.index`/`graph.index`/`state.index` — those are
  named in the source doc's §5.2 list but nothing in `Axiom` today has any
  search or graph/state-transition-history concept to derive them from; they
  are out of scope until a concrete consumer exists (would be genuinely
  speculative infra per `Axiom.SDD/AGENTS.md`'s explicit bootstrap limits).
- No implementation of curated/versioned indexes (`technical-context.index`,
  `skills.index`, `commands.index`, `rules.index`, `roles.index`) — those are
  a different concept per source doc §5.5 and belong to INC-09/INC-10, not
  INC-07.
- No `axiom index validate`/`axiom index diff` (source doc §6.1 lists them
  alongside `rebuild`/`repair`) — out of scope unless the recommendation
  below is accepted and a follow-up increment scopes them explicitly.

## Acceptance criteria

- [x] Confirmed by grep (package names, exported function names, CLI command
      names) whether any `@axiom/index`-equivalent package or `axiom index
      rebuild`-equivalent command exists today in `Axiom`.
- [x] Confirmed whether a direct-scan `list` command already exists for
      increments/bugs/plans (it does not — see Finding 2).
- [x] `@axiom/doctor/src/checks.ts` read in full; confirmed no existing check
      category already validates/rebuilds caches or indexes.
- [x] Confirmed whether `~/.axiom/registry.json` or any INC-01–05 artifact
      has a "regenerate from clean clone" precedent to reuse.
- [x] Recommendation stated and grounded in the findings above, not assumed.
- [x] Recorded in `Axiom.Spec`; no code changed in `Axiom` or `Axiom.SDD`.

## Open questions

None blocking. See "Recommendation" below — it resolves the roadmap's own
open framing question ("is this additive, and is a cache genuinely
low-value") with evidence rather than deferring it.

## Assumptions

- "Direct-scan is fast enough in practice" is evaluated qualitatively
  (current artifact volume in this repo is effectively zero — no
  `axiom.spec/{increments,bugs,plans}/` instances exist yet in `Axiom`'s own
  tree — so there is no real corpus to benchmark against). The recommendation
  below treats this as a reason to defer *building the cache itself* while
  still closing the more urgent, concretely-evidenced gap (see Finding 4).
- `<specPath>` continues to resolve to `<projectRoot>/axiom.spec` per INC-06's
  own stated assumption; this audit does not revisit that resolution
  mechanism.

## Implementation notes

### Finding 1 — No `@axiom/index`, `@axiom/cache`, or `axiom index` command exists anywhere

Grepped `Axiom/packages/*` and `Axiom/apps/*` for `index`, `rebuild`, `cache`
in package names, exported function names, and CLI command/subcommand names:

- No package named `@axiom/index` or `@axiom/cache` exists among the 33
  `@axiom/*` package names in `Axiom/packages/*/package.json`.
- No CLI file registers a top-level `index` command (checked every
  `.command('...')` call across `apps/cli/src/index.ts` and all 34 files in
  `apps/cli/src/commands/`) — commands registered today: `configure`,
  `start`, `audit`, `upgrade`, `components`, `roles`, `topology`,
  `axiom-role`, `axiom-qa-e2e`, `app`, `memory`, `mcp`, `app-plugins*`,
  `intent`, `qa-archive-gate`, `capability`, `context`, `self-update`,
  `sync`, `skills`, `gateway`, `toolchain`, `model`, `app-api`, `join`,
  `repo`, `projects`, `init`, `tui`, `axiom-bug`, `axiom-increment`,
  `axiom-plan`, `doctor`, `axiom-role`. None of these is `index`,
  `rebuild`, or `cache`.
- The only two source-code hits for the string `"rebuild"` in the entire
  repo are inside `Axiom/packages/document-bootstrap/src/
  canonical-agents-md.ts` (plus its generated `dist/` copy and its test) —
  and this is not an implementation. It is a **verbatim string literal**
  reproduced from the external decision document's §18.1 boilerplate, which
  `axiom init`/`configure` writes into every project's generated, canonical
  `AGENTS.md`:

  ```
  If an index appears stale or inconsistent, run:
  - axiom index validate
  - axiom index rebuild
  - axiom doctor
  ```

  **This is a concrete, evidenced gap, not a speculative one**: `Axiom`
  already ships instructions in every generated `AGENTS.md` telling agents
  and users to run `axiom index rebuild`/`axiom index validate`, but neither
  command exists anywhere in the CLI. Any agent or user who follows the
  generated `AGENTS.md`'s own advice today hits a "command not found" — a
  correctness gap in what `Axiom` currently ships, already latent whether or
  not any project has enough artifacts to need a cache.

**Conclusion: confirmed definitively — no index/cache-rebuild concept exists
under any name, in any package, today.** This resolves the roadmap's original
"(unconfirmed — needs audit)" framing to a hard "confirmed absent."

### Finding 2 — There is no direct-scan `list` command either; this is a bigger gap than the roadmap assumed

The roadmap's step 2 asked to check whether direct-scan listing already
works without a cache, since that would reframe "index rebuild" as a
performance/aggregation concern rather than a "listing is impossible at all"
concern.

Read `apps/cli/src/commands/axiom-increment.ts`, `axiom-bug.ts`, and
`axiom-plan.ts` in full (subcommand registration lists, not just names):

- `axiom-increment`: `create`, `refine`, `specify`, `change`, `plan`,
  `plan-approve`, `verify`, `archive`, plus INC-06's additions (`link-plan`,
  `external-ref add|list`). **No `list` subcommand.**
- `axiom-bug`: `create`, `fix-plan`, `verify`, `archive`, plus INC-06's
  additions (`link-plan`, `external-ref add|list`). **No `list` subcommand.**
- `axiom-plan`: `create` (added by INC-06), `approve`, plus INC-06's
  additions (`link-increment`, `link-bug`, `external-ref add|list`). **No
  `list` subcommand.**

Also read `Axiom/packages/workflow/src/artifact-store.ts` in full (all
exported symbols): `resolveSpecRoot`, `resolveArtifactDir`,
`artifactExists`, `loadArtifactMetadata` (single-ID lookup only),
`saveArtifactMetadata`, `ensureArtifactReadme`,
`makeInitialIncrementOrBugMetadata`, `makeInitialPlanMetadata`. **There is
no bulk-list/scan/aggregate function at all** — every read path requires
already knowing the artifact's ID.

**Conclusion: today, there is no way — cached or direct-scan — to answer
"what increments/bugs/plans exist in this project?" from the CLI.** This is
a materially bigger gap than the roadmap's original hypothesis ("index
rebuild would be about performance, not making listing possible at all").
The roadmap's framing assumed a `list` command already existed via direct
folder glob; it does not. **INC-07 is not purely a caching/performance
concern — a minimal `list` capability does not exist yet, cached or
otherwise.**

### Finding 3 — `@axiom/doctor/src/checks.ts` has no cache/index-adjacent check category

Read the file in full (2257 lines). Confirmed check categories today:
`boundaries` (BC-*), `policies` (PC-*), `manifests` (MC-*), `isolation`
(IC-*), `capability-model` (CC-*), `install-profiles` (IP-*), `gateway`
(GW-*), `tool-routing` (TR-*), `topology` (TC-001/002), `qa-lane` (TC-003),
`toolchain`, `memory`, `adapters`, `skills`, `agents`. **None of these reads
or validates `metadata.yml`, `artifact-store.ts`, or any
`.axiom/cache/*.json` concept** — grepped explicitly for
`artifact-store|metadata\.yml|artifactStore|listIncrements|listBugs|
listPlans|glob` inside `checks.ts`: zero matches beyond unrelated strings
(`integrations.yaml`, a code comment about `install-global.mjs`).

The `read → skip-if-absent-or-unparseable → fail/warn/pass with evidence`
pattern is used consistently across every category (e.g. `runManifestChecks`
returns `skip` when `axiom.yaml` can't be read; `runCapabilityModelChecks`
returns `skip` when `providers.yaml` is absent; `runGatewayStateChecks`
returns `skip` when gateway state is uninitialized and not required). This
is exactly the "fallback pattern in spirit" the roadmap referenced, and it
is a genuinely reusable convention for a future `axiom doctor` check that
validates cache freshness (per addendum §5's "axiom doctor debe validar la
coherencia" instruction) — but no such check exists yet.

**Conclusion: confirmed — `@axiom/doctor` implements the right pattern but
has zero coverage of the artifact-store/metadata.yml/cache/index surface
introduced by INC-06.** Any doctor check for cache coherence is itself new
work, not an extension of an existing check.

### Finding 4 — No "regenerate from clean clone" precedent in `@axiom/user-workspace`, but a close analog exists in `context.ts`

Checked `Axiom/packages/user-workspace/src/*.ts` (`registry.ts`,
`registry-types.ts`, `registry-id.ts`, `paths.ts`, `self-update.ts`,
`errors.ts`, `types.ts`) for `rebuild|regenerate|repair|scan|discover`:
**zero matches.** `~/.axiom/registry.json` (INC-01's registry) is a plain
read/write JSON store with `addProject`/`removeProject`/`getProject`/
`listProjects`/`useProject`/`findByRootPath` — it has no self-regeneration
concept; if it were deleted, nothing in the codebase can reconstruct it
(it holds authoring-time-only data: user-registered project root paths),
so it is not a genuine "rebuild from source" analog in the first place
(there is no "source" to rebuild it from — the registry itself is the only
record of where projects live).

However, `apps/cli/src/commands/context.ts`'s `axiom context refresh`
command (0034 Lote C, already shipped) **is** a working, in-repo precedent
for exactly the "derived artifact regenerable from source" shape this
increment needs: `runContextRefresh` re-derives `install-profile.json`
(a generated, non-source-of-truth file under `.sdd/<project>/`) by calling
`installProfile()` fresh and atomically overwriting the file — the same
"derived cache, regenerate on demand, source of truth lives elsewhere"
shape that `axiom index rebuild` would need for
`.axiom/cache/{increments,bugs,plans}.index.json` regenerated from
`metadata.yml` files. `axiom context status` is a parallel read-only
inspector, matching the shape of a future `axiom index validate`.

**Conclusion: `axiom context refresh`/`status` is the closest existing
pattern-match for "rebuild a derived artifact from source, on demand" —
reuse its `run*`/`register*` split and `withProjectContext` wrapper
convention if/when `axiom index rebuild` is built, rather than inventing a
new command-registration style.**

### Reconciling Findings 2 and 4 with the roadmap's original framing

The roadmap asked, in effect: "is direct-scan already fast enough that a
cache is low-value, given INC-06 landed folder-per-artifact?" The premise
behind that question (a working direct-scan `list` already exists) is
false — Finding 2 shows no `list` capability exists at all yet, cached or
otherwise. This changes the shape of the recommendation:

- Building a `.axiom/cache/*.index.json` **cache** ahead of a working `list`
  command would be premature optimization: caching something that isn't
  computed at all yet is exactly the kind of speculative infrastructure
  `Axiom.SDD/AGENTS.md`'s "Explicit Bootstrap Limits" section warns against
  (no mandatory indexes, no complex metadata systems beyond what's asked).
  With zero (or very few) artifact instances in any project today (`Axiom`'s
  own `axiom.spec/{increments,bugs,plans}/` tree does not yet have any
  folder-per-instance data — confirmed no such directories exist in this
  checkout), there is no volume problem to solve yet. A cache optimizes a
  scan that does not exist and has not been shown to be slow.
- Meanwhile, Finding 1's `AGENTS.md`-boilerplate gap is real today,
  independent of artifact volume: every project using `Axiom` already
  receives instructions to run a command that does not exist.

## Validation

`No validation command was found. Performed best-effort validation by inspecting changed files and checking consistency against the requested behavior.`

This increment made no code changes to validate. Best-effort validation
performed:

- Grepped `Axiom/packages/*` and `Axiom/apps/*` directly for `index`,
  `rebuild`, `cache`-related package names, exported symbols, and CLI
  command registrations (`.command('...')` call sites), rather than
  inferring from package names alone.
- Read `Axiom/packages/doctor/src/checks.ts` in full (2257 lines), not a
  partial excerpt, to confirm the absence of any cache/index check category.
- Read `Axiom/packages/workflow/src/artifact-store.ts` in full (all exported
  symbols enumerated) to confirm no bulk-list function exists.
- Read `Axiom/apps/cli/src/commands/axiom-increment.ts`,
  `axiom-bug.ts`, `axiom-plan.ts` subcommand registrations in full to confirm
  no `list` subcommand exists on any of the three.
- Read `Axiom/packages/user-workspace/src/*.ts` for a "regenerate from
  clean clone" precedent (none found) and `Axiom/apps/cli/src/commands/
  context.ts` in full (confirmed a working, reusable "derive from source,
  atomic overwrite" pattern via `runContextRefresh`).
- Re-read both external decision documents' §5 sections (main doc) and §5
  (addendum) directly, not from the roadmap's summary, to ground the
  "gitignored local cache regenerable via `axiom index rebuild`" and "clean
  clone must become operative via `axiom index rebuild`" requirements in
  their exact source wording.

## Result

**No `@axiom/index` package, `@axiom/cache` package, or `axiom index
rebuild`-equivalent command exists anywhere in `Axiom` today, under any
name.** This confirms (rather than assumes) the roadmap's "(unconfirmed —
needs audit)" framing.

**A more significant, previously-unstated gap was found**: there is
currently no way — cached or direct-scan — to list all increments/bugs/plans
in a project. `axiom-increment.ts`/`axiom-bug.ts`/`axiom-plan.ts` have no
`list` subcommand, and `artifact-store.ts` (landed by INC-06) exposes only
single-ID lookups, no bulk enumeration. This means "index rebuild" cannot be
scoped as pure caching/performance work on top of an existing scan — the
scan itself does not exist yet.

Separately, `Axiom`'s own generated `AGENTS.md` boilerplate (verbatim from
the external decision documents' §18.1, materialized by
`@axiom/document-bootstrap`'s `canonical-agents-md.ts`) already instructs
every project to run `axiom index rebuild`/`axiom index validate` when an
index looks stale — commands that do not exist. This is a real, already-
shipping inconsistency, not a speculative future need.

`@axiom/doctor/src/checks.ts`'s `pass/fail/warn/skip` + evidence pattern is
confirmed reusable and has zero existing coverage of the artifact-store/
metadata.yml surface. `apps/cli/src/commands/context.ts`'s `refresh`/
`status` pair (derive-from-source, atomic overwrite, read-only inspector) is
confirmed as the closest existing in-repo precedent for the eventual
`rebuild`/`validate` command shape — reuse its `run*`/`register*` split,
not a new convention.

## Recommendation

**Two-tier recommendation, not a single yes/no:**

1. **Build now, minimal, additive**: a direct-scan `axiom increment list` /
   `axiom bug list` / `axiom plan list` (or a single unified `axiom artifact
   list --kind increment|bug|plan`) that globs
   `<specPath>/{increments,bugs,plans}/*/metadata.yml`, reads each with the
   existing `loadArtifactMetadata`, and prints `id/title/status`. This closes
   Finding 2 (no listing capability exists at all) and is the load-bearing
   prerequisite for anything called "index" to have a meaning — you cannot
   usefully cache a scan that was never implemented. This is genuinely
   additive, small, and does not require inventing a cache file format yet.
2. **Defer the `.axiom/cache/*.index.json` cache itself** (the literal
   `axiom index rebuild` command and its on-disk cache format) until either
   (a) a project has enough artifact instances that the direct-scan `list`
   from step 1 is empirically slow, or (b) a concrete consumer needs the
   cache for something list output alone can't provide (e.g. the future
   `get_implementation_context` MCP tool from INC-14, which the source doc's
   §5.1 rule most directly justifies: "índices derivados = caché local
   regenerable" exists to serve fast programmatic lookups, not human `list`
   output). Building the cache format now, with no consumer and no measured
   scan cost, would be exactly the kind of speculative infrastructure
   `Axiom.SDD/AGENTS.md` prohibits.
3. **Do fix the `AGENTS.md` boilerplate inconsistency (Finding 1) as part of
   whichever increment ships item 1**, either by (a) implementing a minimal
   `axiom index rebuild`/`axiom index validate` as thin wrappers around the
   step-1 `list` scan (report count + basic shape-validity per
   `metadata.yml`, no separate cache file yet — satisfying the literal
   command names the generated `AGENTS.md` promises without committing to a
   cache format), or (b) softening the generated `AGENTS.md` boilerplate to
   not reference commands that don't exist yet, until they do. **Recommend
   (a)**: implementing `axiom index rebuild` as a thin, cache-less
   "re-scan and report" command (using the same read/skip/fail pattern as
   doctor) satisfies both the source doc's literal command name and
   addendum §5's fallback rule ("degradar a scan directo... si no se pueden
   regenerar") in the simplest form — a "rebuild" whose output *is* the
   scan result is a valid, minimal instance of "rebuild," not a cop-out,
   because nothing downstream yet depends on a specific cache file format.
   `axiom doctor` gets a new check category (e.g. `IX-001`) confirming every
   `metadata.yml` under `{increments,bugs,plans}/*/` parses, using the exact
   `pass/fail/warn/skip` convention already established.

This keeps the roadmap's own "no mandatory indexes... unless explicitly
requested" bootstrap limit intact: item 1 is requested (closes a real
listing gap), item 3 closes a real shipped inconsistency, and item 2 (the
literal cache file) is explicitly deferred with a stated, falsifiable
trigger condition rather than built speculatively.

## General spec integration

No integration into a `general-spec.md` was performed — that file does not
exist in this repo (confirmed by every prior spec in the INC-06 chain; the
closest equivalents are `Axiom.Spec/specs/00_Resumen_Ejecutivo.md` through
`08_Glosario.md`, not modified here). This is an audit-only step whose
recommendation may still be adjusted by the next role; consolidating stable
knowledge before that would risk documenting a shape that changes.

## Closure rationale

Per `Axiom.SDD/AGENTS.md`'s closure rules, this increment is set to
`pending`, not `closed`, because:

- Changes were not implemented (by design — this is the migration-engineer
  audit step; audit-only completion is a valid outcome per the roadmap's own
  subagent sequence, but the roadmap's overall INC-07 goal is not yet
  achieved).
- The recommendation above proposes concrete follow-up work (a minimal
  `list` command plus a cache-less `index rebuild`/`index validate`
  reconciling the `AGENTS.md` boilerplate gap); until that work is
  implemented and validated, INC-07 as a whole is not done.

This mirrors the roadmap's own established chain pattern (migration-engineer
audit → next role → validator-reviewer close), not a deviation from it. This
is explicitly NOT proposed as an INC-05-style "audited, no action needed"
closure, because Finding 1 and Finding 2 together identify concrete,
evidenced gaps (a shipping documentation/command mismatch, and a total
absence of any listing capability) that the roadmap's original hypothesis
did not anticipate — closing here would silently drop real findings rather
than act on them.

## Next step recommendation

**index-engineer**, per the roadmap's originally proposed subagent sequence
(migration-engineer → index-engineer → validator-reviewer), now that the
audit is complete. Concrete brief for index-engineer, derived from this
audit rather than assumed:

1. Implement `axiom increment list` / `axiom bug list` / `axiom plan list`
   (or a unified `axiom artifact list --kind <kind>`) as a direct-scan
   reader over `<specPath>/{increments,bugs,plans}/*/metadata.yml`, reusing
   `loadArtifactMetadata` from `@axiom/workflow`'s `artifact-store.ts`
   (already exported from the package barrel). Output: `id`, `title`,
   `status`, `updatedAt` per instance, human-readable by default.
2. Implement `axiom index rebuild` and `axiom index validate` as thin
   wrappers over the same scan from step 1 — `rebuild` re-scans and reports
   what it found (no `.axiom/cache/*.json` file yet, per this audit's
   Recommendation item 2); `validate` re-scans and reports any
   `metadata.yml` that fails to parse (reusing `loadArtifactMetadata`'s
   existing `ArtifactStoreError` discriminated union for the failure
   reasons). Pattern-match `apps/cli/src/commands/context.ts`'s
   `run*`/`register*` split, not a new command-registration convention.
3. Add a new `@axiom/doctor` check category (e.g. `IX-001`,
   category `index` or `artifacts`) that confirms every `metadata.yml`
   under `{increments,bugs,plans}/*/` in the resolved spec scope parses
   correctly, using the exact `pass/fail/warn/skip` + evidence convention
   already established in `checks.ts` (skip cleanly if the spec scope
   doesn't resolve, matching every other check's own skip condition).
4. Do NOT build `.axiom/cache/*.index.json` file persistence, `search.index`,
   `graph.index`, or `state.index` in this pass — explicitly out of scope
   per this audit's Non-goals, to be revisited only when a concrete consumer
   (e.g. INC-14's `get_implementation_context`) or a measured performance
   need justifies it.
5. Hand off to **validator-reviewer** afterward, following the same
   independent-re-verification pattern used to close INC-06 (re-read diffs
   directly, re-run `npm run typecheck`/`build`/`test`, do not trust this
   audit's or index-engineer's prose without independently confirming
   against the actual code).
