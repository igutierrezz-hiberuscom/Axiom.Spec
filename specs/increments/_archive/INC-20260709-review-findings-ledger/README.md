# Increment: Review findings ledger

Status: closed
Date: 2026-07-09

## Goal

Adopt the review-findings-ledger contract, originally designed for a sibling
product (gentle-ai), into Axiom's own review flow. The contract has three
parts: an exhaustive first review pass that keeps sweeping the change under
review until it stops finding anything new, a findings ledger that survives
across a fix-and-re-review cycle instead of being thrown away after each
response, and a scoped re-review that verifies a fix against the ledger
without re-reading the entire original diff from scratch. The contract must
be encoded once, as a single canonical TypeScript string constant, and that
constant must be the thing both the `axiom-reviewer` catalog agent asset and
a drift test refer to, so the two can never quietly drift apart.

## Context

Gentle-ai already solved this problem for its own four review lenses (risk,
readability, reliability, resilience) and for its "judgment-day" adversarial
verification skill. Its two reference documents,
`internal/assets/skills/_shared/review-ledger-contract.md` and
`openspec/specs/review-findings-ledger/spec.md`, describe why a single-pass
review is not enough: each run samples a different subset of the real
issues in a change, so a single read never converges on "everything worth
finding," and a naive re-review after a fix tends to re-surface old issues
as if they were new, because it has no memory of what was already found and
already fixed.

Axiom's situation is simpler in one respect and needs translation in
another. It is simpler because Axiom's bootstrap reviewer today is a single
generalist reviewer (`axiom-review` in the workspace root, and its
product-catalog counterpart `axiom-reviewer`), not four separate lenses, so
the ledger's `lens` field can default to a plain `review` value while still
keeping the door open for dedicated lenses later. It needs translation
because gentle-ai's ledger persistence describes gentle-ai-specific storage
targets (its own `openspec`/`engram`/`none` artifact-store branches, with
gentle-ai-specific paths like `openspec/changes/{change-name}/
review-ledger.md`). Axiom has its own artifact store, built by
`@axiom/workflow`'s `artifact-store.ts`, which persists an increment or a bug
as a folder-per-instance directory under `Axiom.Spec/specs/increments/<ID>/`
or `Axiom.Spec/specs/bugs/<ID>/`, each containing a `README.md` and a
`metadata.yml`. Axiom also has its own Engram-backed memory package,
`@axiom/memory`, whose `MemoryEntry` type already carries an optional
`topicKey` field used for topic-scoped upsert. This increment reads
gentle-ai's two reference documents strictly for the contract's semantics —
the four clauses (exhaustive first pass, findings ledger schema, ledger
persistence, scoped re-review) — and re-expresses them in Axiom-native
wording and Axiom-native storage targets, without copying gentle-ai's prose
or paths verbatim and without modifying anything in the read-only
`C:/repos/GentleAI` repository.

The canonical `AGENTS.md` renderer in
`packages/document-bootstrap/src/canonical-agents-md.ts` already established
the pattern this increment follows: author a piece of shared agent-facing
contract text as a single pure TypeScript string constant, expose a small
renderer function that wraps it in a pair of HTML comment markers
(`AXIOM:GENERATED:START`/`END` in that file's case), and let a drift test
assert that any asset embedding the marker-delimited block stays
byte-identical to what the renderer produces. `tsc -b` compiles `.ts` files
into `dist/`, but it does not copy arbitrary non-`.ts` asset files, which is
why the contract has to live as a string constant inside a `.ts` module
rather than as a separate Markdown asset file that some build step would
need to bundle.

## Scope

- A new module, `packages/document-bootstrap/src/review-ledger-contract.ts`,
  exporting the canonical contract as a pure string constant
  (`REVIEW_LEDGER_CONTRACT`), the list of distinctive phrases the drift test
  checks for (`REQUIRED_LEDGER_CLAUSES`), and a renderer function
  (`renderReviewLedgerContractBlock`) that wraps the constant in a
  `AXIOM:REVIEW-LEDGER-CONTRACT:START`/`END` marker pair.
- Barrel exports of those three symbols from
  `packages/document-bootstrap/src/index.ts`.
- An update to the `axiom-reviewer` catalog agent asset,
  `axiom.spec/target-axiom-agents/axiom-reviewer.md`, adding a new section
  whose body is the marker-delimited block pasted verbatim, plus an update
  to that file's `## Outputs` section noting that the agent now produces a
  findings ledger and a scoped re-review verdict.
- A rewrite of the `## Workflow` and `## Output Contract` sections of the
  workspace-root bootstrap reviewer agent,
  `C:/repos/Axiom Workspace/.claude/agents/axiom-review.md`, so the
  day-to-day reviewer this workspace actually runs adopts the same
  loop-until-dry first pass, ledger persistence, and scoped re-review — in
  its own natural-language wording, since this file is not a drift-test
  target and does not need the machine-readable markers.
- A new drift test,
  `packages/document-bootstrap/tests/review-ledger-contract.test.ts`, that
  fails if the TypeScript constant and the catalog asset are ever edited
  independently of each other.

## Non-goals

- Building four dedicated review lenses (risk, readability, reliability,
  resilience) for Axiom. The schema stays open to them (the `lens` field's
  documented vocabulary includes them), but no dedicated lens agent is
  created in this increment — Axiom's single generalist reviewer keeps using
  the default `review` lens value.
- Building a `judgment-day`-style adversarial verification skill for Axiom.
  That is a gentle-ai-specific construct outside this increment's scope.
- Implementing the actual read/write code that materializes
  `review-ledger.md` inside an artifact folder, or that calls into
  `@axiom/memory`'s Engram backend to upsert a ledger topic. This increment
  encodes the CONTRACT (what an agent following it must do) in agent-facing
  instruction text; it does not add new runtime persistence functions to
  `@axiom/workflow` or `@axiom/memory`. The reviewer agents already have
  file and topic read/write capability through their existing tools; the
  ledger persistence branches direct how they use those tools, they do not
  require new library code.
- Editing `Axiom.Spec/specs/00_Resumen_Ejecutivo.md` through
  `08_Glosario.md`. Any cross-increment consolidation into those files is
  explicitly deferred to the orchestrator's own final pass, not to this
  increment.
- Modifying anything under `C:/repos/GentleAI` or `C:/repos/engram`. Both
  are read-only references for this increment.

## Acceptance criteria

- [x] A single canonical `REVIEW_LEDGER_CONTRACT` string constant exists in
      `packages/document-bootstrap/src/review-ledger-contract.ts` and is
      re-exported from `packages/document-bootstrap/src/index.ts`.
- [x] The `axiom-reviewer` catalog asset
      (`axiom.spec/target-axiom-agents/axiom-reviewer.md`) embeds the
      marker-delimited canonical block, and that embedded block is
      byte-for-byte identical to what `renderReviewLedgerContractBlock()`
      produces.
- [x] The workspace-root bootstrap reviewer workflow
      (`.claude/agents/axiom-review.md`) adopts the loop-until-dry first
      pass, ledger persistence branches, and scoped re-review in its own
      wording.
- [x] A drift test exists and passes, and it is written so that editing
      either the TypeScript constant or the catalog asset alone (without
      updating the other) makes it fail.
- [x] `npm run build` passes.
- [x] The targeted vitest run for `packages/document-bootstrap` passes.

## Open questions

None — decisions baked by orchestrator.

## Assumptions

- Axiom's bootstrap reviewer is a single generalist reviewer today (no 4R
  lens split), so the ledger's `lens` field defaults to `review` while
  keeping the door open, via its documented vocabulary, for dedicated
  `risk`/`readability`/`reliability`/`resilience` lenses if Axiom ever adds
  them.
- "Axiom.Spec artifact store present" is read, for the purposes of this
  contract, as: a folder-per-instance artifact directory already exists for
  the change under review (an increment or bug ID that already has a
  `Axiom.Spec/specs/increments/<ID>/` or `Axiom.Spec/specs/bugs/<ID>/`
  folder). When no such folder exists yet, or the review target is not tied
  to a tracked increment/bug at all (an ad-hoc review of "the current
  change set," for instance), the contract falls through to the Engram
  branch, and then to the in-context-only branch, exactly mirroring
  gentle-ai's own openspec -> engram -> none fallback order translated to
  Axiom's own store names.
- `@axiom/memory`'s `MemoryEntry.topicKey` field is the correct Axiom-native
  analogue of gentle-ai's Engram "topic" concept; both name a stable string
  that a memory backend upserts against, rather than creating duplicate
  entries per write.
- The workspace-root `.claude/agents/axiom-review.md` file is agent-facing
  instruction content, not a canonical spec document, so it may state the
  contract in its own natural, readable prose rather than reproducing the
  machine-verifiable markers — the increment brief was explicit that the
  drift test does not read this file.

## Implementation notes

`packages/document-bootstrap/src/review-ledger-contract.ts` follows the file
header comment style already established by the sibling
`canonical-agents-md.ts` (a banner comment plus a spec-reference line
pointing back at this increment). It exports:

- `REVIEW_LEDGER_CONTRACT` — a single Markdown string with four subsections
  (exhaustive first pass, findings ledger schema as a table, ledger
  persistence honoring Axiom's artifact store, scoped re-review), preceded
  by a short "why this exists" lead line.
- `REQUIRED_LEDGER_CLAUSES` — an array of distinctive, stable substrings
  (things like `` '{LENS}-{NNN}' ``, `'N consecutive'`, `'ceiling'`,
  `'review-ledger.md'`, `'Engram topic'`, `'topicKey'`,
  `'in-context only'`, `'only fix-touched lines'`, `` 'status `info`' ``,
  and the four severities plus the five statuses) that the drift test
  checks are all present in the constant.
- `renderReviewLedgerContractBlock()` — wraps the constant between
  `<!-- AXIOM:REVIEW-LEDGER-CONTRACT:START -->` and
  `<!-- AXIOM:REVIEW-LEDGER-CONTRACT:END -->`, mirroring the
  `AXIOM:GENERATED` marker convention `canonical-agents-md.ts` already uses
  for its own generated block.

`packages/document-bootstrap/src/index.ts` re-exports all three as value
exports, in the same style as the existing
`export { renderCanonicalAgentsMd, writeCanonicalAgentsMd } from
'./canonical-agents-md';` block.

`axiom.spec/target-axiom-agents/axiom-reviewer.md` gained a new
`## Contrato de ledger de hallazgos (review-findings-ledger)` section
whose body is the marker-delimited block pasted verbatim (copied by hand
from `renderReviewLedgerContractBlock()`'s output, character for character),
and its `## Outputs` section now mentions the findings ledger and the
scoped re-review verdict as outputs of the agent.

`.claude/agents/axiom-review.md`'s `## Workflow` section now states the
loop-until-dry first pass (2 consecutive dry sweeps by default, 4-sweep hard
ceiling), the findings-ledger row shape, and the three persistence branches
(artifact folder, Engram topic, in-context-only) in its own words, followed
by a new `## Scoped re-review` section describing the N=1 re-review pass.
The `## Output Contract` section now lists the findings ledger and its
persistence location as explicit outputs.

`packages/document-bootstrap/tests/review-ledger-contract.test.ts` mirrors
the structure of the sibling `canonical-agents-md.test.ts`. Its drift-guard
scenario resolves the repository root the same way the pre-existing
`packages/skills/tests/catalog.test.ts` does: `path.resolve(__dirname, '..',
'..', '..')` from `packages/document-bootstrap/tests/`, landing on the
`Axiom` repository root regardless of the shell's current working
directory, then reads
`axiom.spec/target-axiom-agents/axiom-reviewer.md` from there, extracts the
substring between the two markers (inclusive), and asserts it equals
`renderReviewLedgerContractBlock()`'s output byte-for-byte.

## Validation

From `C:/repos/Axiom Workspace/Axiom`:

- `npm run build` — clean, no errors (`tsc -b` across all workspace
  packages, including the new `review-ledger-contract.ts` module).
- `npx vitest run packages/document-bootstrap` — all test files in
  `packages/document-bootstrap` pass, including the three new scenarios in
  `review-ledger-contract.test.ts` (required-clause coverage, marker
  wrapping, and the drift guard against the `axiom-reviewer` catalog
  asset).

The pre-existing, known baseline failure in
`packages/skills/tests/catalog.test.ts` (missing product-repo root
`axiom.config/skills-catalog.yaml` in this environment) is unrelated to
this increment's scope and was not touched.

## Result

Implemented the review-findings-ledger contract as a single canonical
TypeScript string constant plus a marker-wrapping renderer, embedded that
exact marker-delimited block into the `axiom-reviewer` catalog agent asset,
rewrote the workspace-root bootstrap reviewer's workflow and output
contract to adopt the loop-until-dry first pass, ledger persistence, and
scoped re-review, and added a drift test that fails if the constant and the
asset are ever edited independently. All acceptance criteria are met; build
and targeted tests pass.

## General spec integration

Integrada por el orchestrator en la pasada final cross-increment (batch
INC-20260709-*):

- `04_Flujos_SDD_y_Ciclo_de_Vida.md` — nueva sección "Revisión con ledger de
  hallazgos (loop-until-dry + re-review scoped) — INC-20260709-review-findings-ledger"
  (contrato completo: primera pasada exhaustiva, schema del ledger,
  persistencia artifact-store, re-review scoped).
- `05_Interfaces_Operativas.md` — nueva sección "Reviewer con ledger de
  hallazgos" (superficie del agent `axiom-reviewer`/`axiom-review`).
- `03_Modelo_Operativo_y_Datos.md` — nota sobre el artefacto `review-ledger.md`
  dentro de la carpeta folder-per-artifact del cambio.
- `08_Glosario.md` — término "Ledger de hallazgos (review findings ledger)".

Archivado en `Axiom.Spec/specs/increments/_archive/`.
