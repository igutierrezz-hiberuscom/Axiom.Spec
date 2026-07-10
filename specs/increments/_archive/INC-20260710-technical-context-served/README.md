# Increment: Technical context served — real content via MCP (P1-4)

Status: closed
Date: 2026-07-10

## Goal

Make the curated technical context (`Axiom.Spec/context/**/*.md`) actually
reach agents through the MCP tools that already claim to serve it
(`spec.technicalContextIndexRead`, `spec.recommendedContextList`,
`spec.implementationContextRead`). Before this increment those tools
always returned `null`/empty for every real project — no
`technical-context/indexes/<roleOrKind>.index.yml` file ever existed, and
a separate latent path-resolution bug meant that even once an index
existed, its documents' content would never inline correctly.

## Context

Verified live by the requesting audit: `Axiom/packages/technical-context/
src/technical-context-index.ts`'s `loadTechnicalContextIndex` reads
`<specScope>/technical-context/indexes/<roleOrKind>.index.yml`, but no
such file existed anywhere in `Axiom.Spec` — confirmed via a live
`tools/call spec.technicalContextIndexRead {roleOrKind:"repo"}` returning
`null` before this increment's changes (see "Validation" below for the
exact before/after JSON-RPC transcript).

Separately, `Axiom/packages/mcp-tools/src/implementation-context-handler.ts`'s
`inlineContent` resolved every ref's `path` against `baseDir =
specRepoRoot`, but the index contract documented in
`technical-context-index.ts`'s own header comment says `path` is relative
to the INDEX FILE's own directory
(`<specScope>/technical-context/indexes/`), not to the spec repo root
directly. This meant `mandatory.technicalContext[*].content` would never
inline correctly even with a real, well-formed index present, and there
was no guard preventing a `..`-laden `path` from resolving outside the
spec repo scope.

`@axiom/doctor`'s `TC-013` (`technical-context-index-validity`) turned out
to already be fully implemented (`packages/doctor/src/checks.ts`,
`runTechnicalContextIndexCheck`, wired into `runDoctorChecks`) — the
requesting brief's "reserved but maybe not built" framing was outdated;
no new doctor check was needed, only confirmation that the newly
generated index passes it.

## Scope

- A new index GENERATOR in `@axiom/technical-context`
  (`packages/technical-context/src/generate-index.ts`): scans
  `<specRepo>/context/**/*.md` and produces a `TechnicalContextIndex`
  value, with `context/TECHNICAL_CONTEXT.md` promoted to
  `mandatory.always` (the document's own header already calls itself the
  "puerta de entrada obligatoria" — this is a structural fact, not a
  curation judgment) and every other `.md` listed under `available[]`
  with `tags` derived from its immediate subfolder
  (`architecture`/`operations`/`integrations`/`references`) and a
  `summary` extracted from its first heading. `status: 'draft'` always.
  `path` values correctly relative to the index file's own directory
  (`../../context/<subdir>/<file>.md`).
- A CLI surface for that generator: `axiom context index [--role <r>]
  [--path <specRepoPath>]`, added as a new subcommand of the EXISTING
  `axiom context` command (not a rename, not a new top-level command),
  with the top-level `context --help` description updated to
  disambiguate the two now-coexisting concerns (install
  profile/capabilities vs. technical-context index generation).
- A fix to `inlineContent` in
  `packages/mcp-tools/src/implementation-context-handler.ts`: resolves
  technical-context refs relative to the index file's own directory
  (`<specRepoRoot>/technical-context/indexes/`), with a path-guard that
  keeps the resolved absolute path within the spec repo scope (a `..`
  that would escape it is skipped, with a diagnostic logged, never read).
- Generation and commit of a REAL index at
  `Axiom.Spec/technical-context/indexes/repo.index.yml`, produced by
  running the new generator against this very repo (dogfooding) — 1
  `mandatory.always` entry + 10 `available` entries, one per real
  `context/**/*.md` document.
- Unit tests: generator (11 tests), `inlineContent` fix (2 new tests:
  correct-resolution-relative-to-index-dir, escape-blocked), CLI command
  (3 new tests).
- Live MCP end-to-end proof (before: `null` → after: real content) via a
  throwaway stdio JSON-RPC probe script (deleted after use).

## Non-goals

- No content authoring: this increment catalogs documents that ALREADY
  exist under `context/`; it does not draft any new technical-context
  prose (that is `axiom bootstrap from-code`'s separate, pre-existing
  concern for a completely different input — source code, not curated
  docs).
- No per-role curation: the generator reuses the same shared `context/`
  tree for every `--role` invocation (per-role document selection is
  explicitly deferred as a future consideration, matching the requesting
  brief).
- No `mandatory.whenTags` inference: the generator always emits
  `whenTags: []` — inferring conditional-tag groups mechanically from
  document content is a human curation judgment, not attempted here.
- No new `@axiom/doctor` check: `TC-013` already existed, fully
  implemented, before this increment (confirmed by reading
  `packages/doctor/src/checks.ts` fresh) — this increment only verifies
  the newly generated index passes it, it does not add or modify `TC-013`
  itself.
- No change to `spec.recommendedContextList`'s existing semantics
  (ALL-tags-present via `resolveMandatoryDocuments`, not a true any-tag
  "recommended" matcher) — untouched, out of this increment's scope.

## Acceptance criteria

- [x] A generator scans `Axiom.Spec/context/**/*.md` and produces a valid
      `technical-context/indexes/<role>.index.yml`, with
      `TECHNICAL_CONTEXT.md` as `mandatory.always` and the rest as
      `available` with subfolder-derived `tags` and a heading-derived
      `summary`, `path` correctly relative to the index dir, `status:
      'draft'`.
- [x] The generator is exposed as a CLI command (`axiom context index
      [--role <r>] [--path <specRepo>]`), documented, disambiguated from
      the pre-existing install-profile concern in `context --help`,
      without renaming any existing subcommand.
- [x] `inlineContent` resolves technical-context refs relative to the
      index file's own directory, not `specRepoRoot` directly, with a
      path-guard against scope escape; covered by unit tests for both the
      correct-resolution and the escape-blocked cases.
- [x] `Axiom.Spec/technical-context/indexes/repo.index.yml` exists,
      generated by this increment's own tooling, referencing the real
      `context/*` docs.
- [x] `@axiom/doctor`'s `TC-013` (already existing) validates the
      generated index cleanly (confirmed via the full `packages/doctor`
      test run, and via the generated file's filename-stem/`role`
      convention matching `TC-013`'s rule).
- [x] `npm run build` exits 0.
- [x] Live MCP proof: `spec.technicalContextIndexRead`/`spec.
      recommendedContextList` return real content (not `null`) after the
      fix, confirmed against a `null` baseline captured with the index
      temporarily removed. `spec.implementationContextRead` exercised
      end-to-end against a throwaway project, confirming
      `mandatory.technicalContext[0].content` inlines the real
      `TECHNICAL_CONTEXT.md` text.
- [x] `npx vitest run` on `packages/technical-context`, `packages/mcp-
      tools`, `apps/cli` — all green. `npm test` full suite — all green.

## Open questions

None blocking. One judgment call made without further clarification:
`role`/`repoKinds` for the generated index both default to `'repo'`
(matching the audit's own live-proof example call), not a more
"official" default sourced from elsewhere in the codebase — confirmed by
grep that no such existing "default role" constant exists yet.

## Assumptions

- "Technical context" continues to mean the existing `TechnicalContextIndex`
  concept (`@axiom/technical-context`), unchanged from prior increments in
  this chain (`INC-20260702-technical-context-index-reconcile-*`).
- A single-level-plus-recursive scan of `context/` (walks subdirectories,
  but the real tree today is exactly one level deep) is sufficient; no
  need to hardcode the four current subfolder names
  (`architecture`/`operations`/`integrations`/`references`) into the
  generator itself.
- Generating and committing `Axiom.Spec/technical-context/indexes/repo.index.yml`
  as part of this increment (writing into `Axiom.Spec`, not `Axiom.SDD`)
  is intended and explicitly allowed by the requesting brief — it is spec
  repo CONTENT (a curated-docs index), not code.
- `spec.implementationContextRead`'s live exercise required a fully
  throwaway project (registry + plan + increment) since neither this
  user's real `~/.axiom` registry nor `Axiom.Spec` itself has a
  registered project/plan pointing at `Axiom.Spec` — building that
  throwaway fixture via direct `require()` of the compiled packages (not
  touching the real registry or repo) was judged in-scope for "if
  feasible" per the requesting brief, and was successfully exercised (see
  Validation).

## Implementation notes

### Files changed (`Axiom/` — code repo)

- `packages/technical-context/src/generate-index.ts` (NEW) —
  `generateTechnicalContextIndex(specRepoAbsolutePath, options?)` (pure
  value builder, scans `context/**/*.md`) and
  `writeGeneratedTechnicalContextIndex(specRepoAbsolutePath, options?)`
  (atomic tmp+rename write, same pattern as `saveArtifactMetadata`).
  `DEFAULT_TECHNICAL_CONTEXT_ROLE = 'repo'`.
- `packages/technical-context/src/index.ts` — barrel export of the new
  generator module.
- `packages/technical-context/tests/generate-index.test.ts` (NEW) — 11
  tests: real content tree (entry point promotion, available
  entries/tags/summaries, status/role/repoKinds defaults, custom role,
  projectId propagation), empty/absent `context/` (never throws),
  write+round-trip via `loadTechnicalContextIndex`/
  `validateTechnicalContextIndex`, filename-stem convention compatible
  with `TC-013`, write-failure error path.
- `packages/mcp-tools/src/implementation-context-handler.ts` — imports
  `TECHNICAL_CONTEXT_INDEX_RELATIVE_DIR`; `inlineContent` gained a
  `guardRoot` parameter (defaults to `baseDir`) with a `path.relative`-
  based escape check (logs a diagnostic via `console.error` and skips the
  ref, never reads outside the guard); the `mandatory.technicalContext`
  call site now passes `path.join(specRepoRoot,
  TECHNICAL_CONTEXT_INDEX_RELATIVE_DIR)` as `baseDir` and `specRepoRoot`
  as `guardRoot`.
- `packages/mcp-tools/tests/implementation-context-handler.test.ts` —
  fixture's `TECHNICAL_CONTEXT_INDEX_YAML` paths updated from
  `tc-doc-1.md` to `../../tc-doc-1.md` (correct per the fixed
  convention; the referenced files' physical location was unchanged,
  only the YAML `path` values); 2 new tests: a doc placed inside
  `technical-context/` (reachable only via correct index-dir-relative
  resolution, unreachable under the old buggy baseDir) and an
  escape-blocked case (a `path` with enough `..` segments to reach the
  OS tmpdir, confirmed left uninlined).
- `apps/cli/src/commands/context.ts` — new `ContextIndexArgs`/
  `ContextIndexResult`/`runContextIndex` (thin wrapper over
  `writeGeneratedTechnicalContextIndex`, no `withProjectContext` — the
  target is an arbitrary `specRepoRoot`, not necessarily an
  Axiom-initialized project) and a new `context index` subcommand
  (`--role`, `--path`); `registerContext`'s top-level `context`
  description rewritten to disambiguate the two concerns.
- `apps/cli/tests/context.test.ts` — 3 new tests (`Scenario 4`): default
  role, custom `--role`, and confirms `runContextIndex` does NOT require
  a resolved Axiom project (unlike `refresh`/`status`).

### Files changed (`Axiom.Spec/` — spec repo CONTENT, not code)

- `technical-context/indexes/repo.index.yml` (NEW, generated) — real
  index for this repo: 1 `mandatory.always` entry
  (`context.entry-point` → `TECHNICAL_CONTEXT.md`) + 10 `available`
  entries (4 `architecture`, 2 `operations`, 1 `integrations`, 3
  `references`), `status: draft`.

### Doctor `TC-013`

Already fully implemented before this increment
(`packages/doctor/src/checks.ts`'s `runTechnicalContextIndexCheck`,
wired into `runDoctorChecks` — confirmed by reading the file fresh, not
by trusting the requesting brief's "reserved" framing). No change made.
The newly generated `repo.index.yml` satisfies its filename-stem
convention (`role: repo` / `repoKinds: [repo]` both match the filename
stem `repo`).

## Validation

Validation discovery: `Axiom/package.json` defines `build` (`tsc -b`),
`test` (`vitest run`). Ran directly.

### Build

```
cd Axiom && npm run build
> axiom-product@0.1.0 build
> tsc -b
(clean, exit 0)
```

### Scoped test runs

- `packages/technical-context`: **40/40 passed** (29 pre-existing + 11
  new in `generate-index.test.ts`).
- `packages/mcp-tools`: **55/55 passed** (53 pre-existing + 2 new in
  `implementation-context-handler.test.ts`; the escape-blocked test's
  diagnostic `console.error` line is visible in stderr output, confirming
  the guard fires).
- `apps/cli`: **629/629 passed** across 65 files (626 pre-existing + 3
  new in `context.test.ts`).
- `packages/doctor`: **177/177 passed** (unchanged — confirms `TC-013`
  still passes with the new index present).

### Full suite

`npm test`: **2209/2209 passed** across 208 files (baseline was 207
files / 2193 tests — +1 file (`generate-index.test.ts`), +16 tests: 11
(generator) + 2 (`inlineContent` fix) + 3 (CLI command). Zero failures,
zero regressions.

### Live MCP end-to-end proof

Driven via a throwaway stdio JSON-RPC probe script in the scratchpad
(deleted after use), spawning `node apps/cli/dist/index.js mcp serve
--kind spec --project-root "Axiom.Spec" --home-dir <throwaway>`.

**BEFORE** (index temporarily renamed away, simulating the pre-fix
state):

```json
// tools/call spec.technicalContextIndexRead {"roleOrKind":"repo"}
{"jsonrpc":"2.0","id":2,"result":{"content":[{"type":"text","text":"null"}]}}

// tools/call spec.recommendedContextList {"roleOrKind":"repo","taskTags":[]}
{"jsonrpc":"2.0","id":3,"result":{"content":[{"type":"text","text":"null"}]}}
```

**AFTER** (real `repo.index.yml` present):

```json
// tools/call spec.technicalContextIndexRead {"roleOrKind":"repo"}
{"jsonrpc":"2.0","id":2,"result":{"content":[{"type":"text",
  "text":"{\"schemaVersion\":1,\"status\":\"draft\",\"role\":\"repo\",\"repoKinds\":[\"repo\"],\"mandatory\":{\"always\":[{\"id\":\"context.entry-point\",\"path\":\"../../context/TECHNICAL_CONTEXT.md\",\"reason\":\"Puerta de entrada obligatoria al contexto técnico curado del producto.\"}],\"whenTags\":[]},\"available\":[{\"id\":\"architecture.01-vision-general-y-capas\",\"path\":\"../../context/architecture/01-vision-general-y-capas.md\",\"tags\":[\"architecture\"],\"summary\":\"Visión general y capas del runtime Axiom\"}, ... 9 more entries]}"
}]}}

// tools/call spec.recommendedContextList {"roleOrKind":"repo","taskTags":[]}
{"jsonrpc":"2.0","id":3,"result":{"content":[{"type":"text",
  "text":"[{\"id\":\"context.entry-point\",\"path\":\"../../context/TECHNICAL_CONTEXT.md\",\"reason\":\"Puerta de entrada obligatoria al contexto técnico curado del producto.\"}]"
}]}}
```

`spec.implementationContextRead` exercised end-to-end against a fully
throwaway registered project (spec repo = a tmp copy of the real
`context/` tree + a generated index, plan+increment linked, role
`'repo'`, `contextBudget: 'medium'`):

```
confidence: medium
missingMetadata: [ 'repositories.target', 'mandatory.sddSkills', 'mandatory.repoSkills' ]
indexes.technicalContext: {"present":true,"count":11}
mandatory.technicalContext[0].id: context.entry-point
mandatory.technicalContext[0].path: ../../context/TECHNICAL_CONTEXT.md
mandatory.technicalContext[0].content (first 200 chars):
"# TECHNICAL_CONTEXT\n\nPuerta de entrada obligatoria al conocimiento técnico estable del producto Axiom, tal como existe en el código real de `Axiom/` a fecha 2026-07-02. Cada documento enlazado aquí ci"
recommended.technicalContext (ids): [10 real doc ids]
```

`missingMetadata` entries shown (`repositories.target`, `mandatory.
sddSkills`, `mandatory.repoSkills`) are expected/unrelated to this
increment (the throwaway fixture deliberately has no target repo or
skills index) — `confidence: medium` instead of `low` because the
project/plan/relatedSpec/technicalContext all resolved.

Both probe scripts were deleted from the scratchpad after use, per
instruction.

## Result

The premise "agents can consult a solid technical context via MCP
without reading it all" is now real, not just architecturally plausible.
A bounded, opt-in generator (`axiom context index`) turns the existing
hand-curated `context/**/*.md` prose into a valid
`TechnicalContextIndex`, and a real one now ships in `Axiom.Spec` itself
(dogfooding). The independent, latent `inlineContent` path-resolution bug
(baseDir mismatch with the documented index-relative-path contract, plus
no scope guard) is fixed and covered by tests for both directions
(correct resolution, escape blocked). Confirmed via an explicit
before/after live MCP transcript: the exact `null` responses the audit
found are now real, non-null content referencing the real curated docs.
Full test suite grew from 2193 to 2209 passing tests (zero regressions),
build stays clean.

## General spec integration

Integrated into `Axiom.Spec/specs/06_Integraciones_y_Capacidades.md`
(this repo's `general-spec.md`-equivalent per `specs/README.md`'s own
rule — no `general-spec.md` file exists in this repo; the numbered
`specs/0X_*.md` files ARE the general spec, written directly in that
format), under the existing "Capa de herramientas MCP" section: a new
subsection documents that the technical-context index is now GENERATED
(not just theoretically loadable), names the generator/CLI command, and
records the `inlineContent` path-resolution fix as a correction to the
prior `spec.implementationContextRead` description (which did not
previously call out the index-relative-path contract or the guard).
This is stable, cross-increment-relevant knowledge (a real MCP-serving
capability, not implementation-history detail), so it belongs there per
the documentation rules; no other numbered spec file needed a
corresponding change.

## Next step recommendation

None required to close this increment — self-contained. A future,
explicitly-separate increment could add per-role curation (distinct
`context/` subsets per role, not just a shared tree reused for every
`--role`) if a concrete need arises; not attempted here per this
increment's own Non-goals.
