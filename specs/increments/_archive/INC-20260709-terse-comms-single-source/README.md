# Increment: Terse comms single source

Status: closed
Date: 2026-07-09

## Goal

De-duplicate the inter-agent terse-comms guidance — the symbol legend and the
"never compress anything human-facing" rules — from two hand-maintained
copies into a single bundled TypeScript constant that both consuming
renderers embed verbatim, guarded by a drift test that fails the moment the
two surfaces diverge.

## Context

INC-20260708-caveman-terse-comms-skill introduced this guidance in two
places at once, deliberately kept "in sync" by convention rather than by
construction:

1. `packages/document-bootstrap/src/canonical-agents-md.ts`'s
   `renderInterAgentCommsBlock()` — the `## Inter-agent communication style`
   section of the canonical, repo-root `AGENTS.md`. English.
2. `apps/cli/src/commands/workspace-skills.ts`'s
   `CANONICAL_SEED_SOURCES['axiom-terse-comms']` — the body of the bundled
   `axiom-terse-comms` seed skill. Spanish framing, with its own
   hand-authored "Regla dura: qué NUNCA se comprime" and "Legend de símbolos
   (subset curado)" sections that happened to restate the same rules and the
   same seven symbols in different words.

Nothing enforced that the two stayed identical; a future edit to one without
the other would silently drift the two surfaces apart, and an agent
following only one of the two paths (AGENTS.md without ever applying the
skill, or vice versa) would see a subtly different rule set. This is the
same class of problem `canonical-agents-md.ts` and
`review-ledger-contract.ts` already solved for their own boilerplate blocks:
author the shared text once, as a pure string constant, and let a drift
test — not developer discipline — keep every consumer honest.

## Scope

- A new module, `packages/document-bootstrap/src/inter-agent-comms.ts`,
  exporting the shared legend + rules as pure string constants
  (`INTER_AGENT_TERSE_RULES`, `INTER_AGENT_SYMBOL_LEGEND`, and a convenience
  concatenation `INTER_AGENT_COMMS_LEGEND_AND_RULES`), with a canonical
  English wording that reproduces `renderInterAgentCommsBlock()`'s
  pre-existing text exactly.
- Barrel exports of those three symbols from
  `packages/document-bootstrap/src/index.ts`.
- A refactor of `renderInterAgentCommsBlock()` in `canonical-agents-md.ts` to
  build its output from `INTER_AGENT_COMMS_LEGEND_AND_RULES` instead of an
  inline string-literal array. Purely internal: the rendered output is
  byte-identical to before.
- A rebuild of `CANONICAL_SEED_SOURCES['axiom-terse-comms']` in
  `apps/cli/src/commands/workspace-skills.ts` so its body embeds
  `INTER_AGENT_TERSE_RULES` and `INTER_AGENT_SYMBOL_LEGEND` verbatim (via a
  package import of `@axiom/document-bootstrap`) around a short, unchanged
  Spanish framing (title, "Cuándo usarla", the "Guía breve" heading and its
  short bullet list of style tips that is not itself duplicated guidance).
- A new named export, `AXIOM_TERSE_COMMS_SEED_SOURCE`, exposing that body so
  a drift test can read it without reaching into the module's private
  `CANONICAL_SEED_SOURCES` map.
- A new drift test, `apps/cli/tests/terse-comms-drift.test.ts`, asserting
  both surfaces contain the shared constants verbatim, and that the symbol
  legend and the never-compress rules each appear exactly once per surface
  and match the shared constant byte-for-byte — so a future hand-edit that
  re-inlines a divergent copy into either surface fails this test.

## Non-goals

- Changing the meaning or coverage of the guidance itself (the scope
  statement, the never-compress list, the seven-symbol legend, or the
  code/paths/commands/error-strings clause). This increment is a
  de-duplication refactor, not a content change.
- Changing `renderInterAgentCommsBlock()`'s observable output. The existing
  26 tests in `packages/document-bootstrap/tests/canonical-agents-md.test.ts`
  must keep passing unchanged.
- Translating the shared constant into Spanish for the seed skill. The
  brief explicitly accepts the embedded sections staying in English
  (agent-facing content), so the single source does not fork by language.
- Editing `Axiom.Spec/specs/00_Resumen_Ejecutivo.md` through
  `08_Glosario.md`. Any cross-increment consolidation into those files is
  deferred to the orchestrator's own final pass.
- Adding new tsconfig project references or package.json dependencies
  beyond what the existing build graph already resolves (see Implementation
  notes — none were needed).

## Acceptance criteria

- [x] One shared TypeScript constant file
      (`packages/document-bootstrap/src/inter-agent-comms.ts`) is the single
      source of the legend + rules, and is barrel-exported from
      `packages/document-bootstrap/src/index.ts`.
- [x] `renderInterAgentCommsBlock(true)`'s output is unchanged: all 26
      pre-existing tests in `canonical-agents-md.test.ts` pass without
      modification.
- [x] The `axiom-terse-comms` seed skill embeds the shared constant(s)
      verbatim, and the seed source is exported
      (`AXIOM_TERSE_COMMS_SEED_SOURCE`) so a test can assert on it.
- [x] A drift test exists, passes, and fails if the two surfaces diverge
      (verified by construction: it asserts exact, single-occurrence,
      byte-identical embedding on both sides).
- [x] `npm run build` passes; the targeted vitest run passes.

## Open questions

None — decisions baked by orchestrator.

## Assumptions

- `workspace-skills.ts` is compiled directly by `apps/cli`'s own
  `tsconfig.json` (it is not one of the files `@axiom/cli-commands` claims
  via its `rootDir: "../.."` + explicit `include` list — that list only
  names `configure.ts`, `sync.ts`, `upgrade.ts`, `_shared.ts`, `model.ts`,
  `components.ts`, `index-cmd.ts`, `validate-changes.ts`, `repair.ts`,
  `toolchain.ts`, and `mcp.ts`). `apps/cli/tsconfig.json` already has a
  `@axiom/document-bootstrap` path mapping and project reference, and
  `apps/cli/package.json` already lists `@axiom/document-bootstrap` as a
  dependency, so importing it from `workspace-skills.ts` needed no build
  graph changes. The increment brief's build-resolution note anticipated a
  possible `@axiom/cli-commands` single-ownership conflict; that conflict
  does not apply to this specific file, and the build passed with zero
  tsconfig/package.json edits.
- It is acceptable for the `axiom-terse-comms` seed skill's embedded legend
  + rules to read in English inside an otherwise-Spanish document, because
  the content is agent-facing (not human-facing prose subject to the
  terse-comms rules' own "never compress anything a human reads" clause).
- Preserving the pre-existing `apps/cli/tests/workspace-skills.test.ts`
  assertions (`'agente-a-agente'`, `'NUNCA'`, `'Mensajes de commit'`,
  `'READMEs de incremento'`, `'→'`, `'∀'`, `'⊥'`) without editing that test
  file (out of this increment's scope) is a hard constraint. The rebuilt
  seed body keeps a short Spanish lead-in sentence, immediately before the
  embedded shared constants, that legitimately mentions the same categories
  ("Mensajes de commit y de PR, READMEs de incremento/bug...") rather than
  re-enumerating the full never-compress list a second time — the full,
  canonical list is the embedded shared constant; the lead-in is scoping
  prose, not a second inventory of items.

## Implementation notes

`packages/document-bootstrap/src/inter-agent-comms.ts` follows the banner
comment style already established by `canonical-agents-md.ts` and
`review-ledger-contract.ts`. It exports three pure string constants with no
placeholders and no interpolation:

- `INTER_AGENT_TERSE_RULES` — the scope statement ("Terse,
  symbol-legend-backed shorthand is acceptable ONLY for
  Axiom-agent-to-Axiom-agent traffic...") plus the never-compress list
  (spec docs 00-08, increment/bug READMEs, decision docs, PR/commit
  messages, user-facing summaries, generated documentation).
- `INTER_AGENT_SYMBOL_LEGEND` — the seven-symbol legend (`→ ∀ ! ⊥ ? ~
  ok/fail`) plus the closing "code, paths, commands and error strings are
  never compressed" clause.
- `INTER_AGENT_COMMS_LEGEND_AND_RULES` — the two above, joined with the same
  blank-line spacing `renderInterAgentCommsBlock()` already used, so callers
  that want the whole body in one shot do not have to re-derive the join.

`canonical-agents-md.ts`'s `renderInterAgentCommsBlock()` now returns
`['## Inter-agent communication style', '', INTER_AGENT_COMMS_LEGEND_AND_RULES].join('\n')`
instead of a 15-line inline string-literal array. The join arithmetic was
checked by hand against the original array-of-lines `.join('\n')` — an empty
array element between two array elements produces exactly the same
double-newline paragraph break as `'\n\n'` inside a template literal — and
confirmed byte-identical by running the existing 26-test suite unchanged.

`workspace-skills.ts` now imports `INTER_AGENT_TERSE_RULES` and
`INTER_AGENT_SYMBOL_LEGEND` from `@axiom/document-bootstrap` (a package
import, mirroring its existing `@axiom/skills` import — never a relative
path into the package's own `src/`, respecting the
INC-20260703-cli-commands-single-ownership layering rule). The
`axiom-terse-comms` seed body keeps its `# Axiom Terse Comms` title, its
intro paragraph, and its `## Cuándo usarla` section unchanged, then under a
single `## Guía breve` heading: a short Spanish lead-in sentence, the
embedded `INTER_AGENT_TERSE_RULES`, the embedded `INTER_AGENT_SYMBOL_LEGEND`,
and a trimmed three-bullet style-tips list (the previous fourth bullet,
"nunca comprimas código, paths, comandos, ni strings de error," was dropped
because it is now covered verbatim by the embedded legend's own closing
clause — keeping it would have been exactly the re-duplication this
increment removes). The pre-existing standalone `## Regla dura: qué NUNCA se
comprime` and `## Legend de símbolos (subset curado)` headings were removed;
their content is now the embedded shared constants.

`AXIOM_TERSE_COMMS_SEED_SOURCE` is exported immediately after the
`CANONICAL_SEED_SOURCES` map as
`CANONICAL_SEED_SOURCES['axiom-terse-comms']`, a narrow named export rather
than exposing the whole map, per the brief's stated preference.

`apps/cli/tests/terse-comms-drift.test.ts` mirrors the structure of
`packages/document-bootstrap/tests/review-ledger-contract.test.ts`: it
imports `renderCanonicalAgentsMd` and the two shared constants from
`@axiom/document-bootstrap`, and `AXIOM_TERSE_COMMS_SEED_SOURCE` from
`../src/commands/workspace-skills` (relative import, same style as the
pre-existing `apps/cli/tests/workspace-skills.test.ts`). It asserts, in
order: (1) each shared constant is `.toContain`-present in
`renderInterAgentCommsBlock(true)`'s output; (2) each shared constant is
`.toContain`-present in the exported seed source; (3) a drift guard —
using a small local `countOccurrences` helper — that each shared constant
occurs exactly once in each surface, and that the extracted substring at
that occurrence is `.toBe`-identical (not just `.toContain`) to the shared
constant. A future edit that re-inlines a hand-tweaked copy of the legend
or the rules into either surface, alongside or instead of the import, would
either fail the "exactly once" count (two near-identical copies present) or
the byte-identical extraction (one divergent copy present) — either failure
mode this test catches.

No new tsconfig `references` or `package.json` dependencies were needed:
`apps/cli/tsconfig.json` already had both the `@axiom/document-bootstrap`
path mapping and project reference, and `apps/cli/package.json` already
listed `@axiom/document-bootstrap` as a dependency (used elsewhere in
`apps/cli/src` already). `workspace-skills.ts` is not part of the
`@axiom/cli-commands` single-ownership `include` list, so no cli-commands
project reference was needed either.

## Validation

From `C:/repos/Axiom Workspace/Axiom`:

- `npm run build` — clean, no errors (`tsc -b` across all workspace
  packages, including the new `inter-agent-comms.ts` module and the edited
  `workspace-skills.ts`).
- `npx vitest run packages/document-bootstrap apps/cli/tests/terse-comms-drift.test.ts apps/cli/tests/workspace-skills.test.ts`
  — 8 test files, 88 tests, all passing: the pre-existing
  `canonical-agents-md.test.ts` (26 tests, unchanged), the pre-existing
  `workspace-skills.test.ts` (17 tests, unchanged), and the new
  `terse-comms-drift.test.ts` (8 tests).
- `packages/skills/tests/catalog.test.ts` (the known baseline reference in
  the brief) was also run in isolation as a sanity check: it currently
  passes in this environment (11/11), so there was no pre-existing baseline
  failure to classify against, and no newly-introduced failure either.

## Result

Extracted the inter-agent terse-comms legend and never-compress rules into
one shared TypeScript module, `packages/document-bootstrap/src/
inter-agent-comms.ts`, consumed verbatim by both `renderInterAgentCommsBlock()`
(byte-identical output, all 26 pre-existing tests pass unchanged) and the
bundled `axiom-terse-comms` seed skill (all 17 pre-existing tests pass
unchanged, including the ones asserting on Spanish-language substrings kept
in the seed's framing). Added a drift test asserting exact,
single-occurrence, byte-identical embedding on both sides. Build is clean;
no tsconfig or package.json changes were required. All acceptance criteria
are met.

## General spec integration

Integrada por el orchestrator en la pasada final cross-increment (batch
INC-20260709-*):

- `06_Integraciones_y_Capacidades.md` — addendum a la subsección "Skill de
  comunicación terse inter-agente" documentando la fuente única
  (`inter-agent-comms.ts`) consumida por ambos renderers + el test de drift.
- `08_Glosario.md` — término "Fuente única de terse-comms
  (`inter-agent-comms.ts`)".

Archivado en `Axiom.Spec/specs/increments/_archive/`.
