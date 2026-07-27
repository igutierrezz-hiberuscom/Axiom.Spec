# Increment: adapter-notes-fastfollow

Status: closed
Date: 2026-07-26

## Goal

Fix stale/misleading adapter-note text that the same batch made
inconsistent: INC-2 (`INC-20260726-adapter-generators`) gave
`codex`/`antigravity`/`visual-studio-2026` real generator packages whose
bundled adapter note said "Axiom does NOT write a native MCP file for
this target; verified native MCP projection comes in a later increment."
INC-3 (`INC-20260726-adapter-mcp-parity`) then made that false: it writes
`.vs/mcp.json` for `visual-studio-2026` and emits an accurate
user-global MCP note (via `native-mcp-config.ts`) for `codex`/
`antigravity`. This fast-follow re-aligns the generator-emitted notes
(and one CLI doc comment) with that new reality so an agent reading a
generated `.vs/AXIOM.md`/`.antigravity/AGENTS.md`/`.codex/AGENTS.md`
gets accurate information instead of being told MCP isn't configured
when it now is.

## Context

Batch order this increment depends on:
1. `INC-20260726-adapter-generators` — gave codex/antigravity/
   visual-studio-2026 real single-file generators (mirroring
   claude-code), each emitting a static adapter-specific note baked in
   at generation time.
2. `INC-20260726-adapter-mcp-parity` — added `.vs/mcp.json` writing for
   `visual-studio-2026` (`native-mcp-config.ts`, DOCUMENTED ASSUMPTION,
   same shape as VS Code) and accurate USER-GLOBAL informative MCP notes
   for `codex`/`antigravity` (`buildUserGlobalMcpNote`), without
   touching the static notes baked into the generator packages from (1).

## Scope

- `packages/adapters/visual-studio-2026/src/agents-md.ts`:
  `VISUAL_STUDIO_2026_ADAPTER_NOTE` + its doc comment — now states Axiom
  writes `.vs/mcp.json` (VS solution-level MCP,
  `{ servers: { <id>: { type:'stdio', ... } } }`), a documented,
  override-able assumption for VS 2022 17.14+/2026, carrying
  `sdd-mcp-server` + `spec-mcp-broker`; manual fallback if the VS build
  doesn't read that file.
- `packages/adapters/antigravity/src/agents-md.ts`:
  `ANTIGRAVITY_ADAPTER_NOTE` + its doc comment — states antigravity's MCP
  config is USER-GLOBAL (`~/.gemini/config/mcp_config.json`,
  `mcpServers`), no project file written, aligned with
  `native-mcp-config.ts`'s wording.
- `packages/adapters/codex/src/agents-md.ts`: `CODEX_ADAPTER_NOTE` + its
  doc comment — now also states codex's MCP config is USER-GLOBAL
  (`~/.codex/config.toml`, `[mcp_servers]`), no project file written
  (previously said nothing about MCP at all).
- `apps/cli/src/commands/init.ts`: `ADAPTER_TARGETS` doc comment — no
  longer claims vscode/codex are thin/fallback with "the real generator
  comes in a later increment"; states both have first-class generators
  and describes each one's actual MCP treatment (`vscode`
  project-scoped/VERIFIED, `codex` user-global/informative).

## Non-goals

- No behavior change: no new files written, no new writer logic. Text
  only.
- No change to `native-mcp-config.ts` itself (already accurate from
  INC-3).
- Left `packages/installer/src/registry.ts` (`codex` entry comment) and
  `packages/model-routing/src/support-matrix.ts` (`codex` entry comment)
  untouched: found during a grep sweep, both still say
  "sin MCP dedicado todavía (llega en un incremento posterior)" for
  codex, which is now slightly stale in the same way, but they were not
  part of the requested scope and are lower-stakes (internal source
  comments, not agent-facing generated content). Flagged here as a
  possible future micro-fast-follow, not fixed in this increment to keep
  the change tight and focused.
- Left `apps/cli/src/commands/workspace-adapters.ts`'s
  `ANTIGRAVITY_EXTRA_NOTE`/`VISUAL_STUDIO_2026_EXTRA_NOTE`/
  `CODEX_EXTRA_NOTE` untouched: verified they are intentional, clearly
  labelled dead code (no caller since INC-2 gave these 3 targets
  dedicated generators) with their own doc comment explaining the
  content was folded into the dedicated packages — not misleading to an
  agent since nothing reads them anymore.
- Left `Axiom.Spec/context/architecture/04-adapters-y-model-routing.md`
  untouched even though it is also now stale in the same family (its
  "Adapters con package dedicado (6)" table and "Targets declarados sin
  adapter dedicado" section predate INC-2/INC-3 entirely — it doesn't
  list `codex`/`antigravity`/`visual-studio-2026` as having real
  generator packages, and says nothing about native MCP writing).
  Fixing it properly means rewriting a substantial chunk of that doc,
  not a small text fix, so it's out of scope for this tight fast-follow.
  Flagged here as a recommended follow-up increment (this repo has no
  `general-spec.md`; this architecture doc is its closest equivalent for
  the adapters/model-routing domain).

## Acceptance criteria

- [x] `visual-studio-2026` adapter note accurately describes `.vs/mcp.json`
      writing (shape, servers, override-able assumption, manual
      fallback).
- [x] `antigravity` adapter note accurately describes the USER-GLOBAL MCP
      note behavior, consistent with `native-mcp-config.ts`'s wording.
- [x] `codex` adapter note accurately describes the USER-GLOBAL MCP note
      behavior, consistent with `native-mcp-config.ts`'s wording.
- [x] `apps/cli/src/commands/init.ts`'s `ADAPTER_TARGETS` doc comment no
      longer claims vscode/codex lack real generators/MCP treatment.
- [x] Existing generator tests (`generator.test.ts` x3) and workspace
      adapter tests (`workspace-adapters.test.ts`,
      `adapters.e2e.test.ts`) still pass — none needed edits because the
      new wording was crafted to preserve every asserted substring
      (`'no fue re-verificado'`, `'mcp_config.json'`, `'mcpServers'`,
      `'Codex CLI lee este archivo'`, the `## <Target> adapter notes`
      headings).
- [x] `npm run build` passes.

## Open questions

None — scope was fully specified by the requesting brief.

## Assumptions

- The `.vs/mcp.json` DOCUMENTED ASSUMPTION framing (VS 2022 17.14+/2026,
  not independently re-verified) carries over unchanged from
  `native-mcp-config.ts`; this increment only makes the adapter-note
  wording consistent with it, it does not re-verify or change the
  assumption itself.

## Implementation notes

Edited 4 files (see Scope). Each adapter note's preceding doc comment
was also updated (not just the note string) so the "why" stays
accurate for future readers — the old comments explicitly said "native
MCP projection is a later increment (INC-3), not this one," which was
the exact staleness being fixed.

New `VISUAL_STUDIO_2026_ADAPTER_NOTE` (paste, Spanish, house style):

> Visual Studio 2026 (gateway-first enterprise IDE) delega en este
> archivo (`.vs/AXIOM.md`) + en la superficie portable `.axiom/`
> (skills/rules/commands/agents) y el workflow SDD canónico de Axiom
> (spec aprobada -> plan aprobado -> decisiones explícitas (ADR) ->
> código real -> manifests canónicos) descrito en el preamble de este
> archivo.
>
> Axiom escribe config MCP nativo en `.vs/mcp.json` (MCP a nivel
> solución de Visual Studio, mismo shape que VS Code:
> `{ servers: { <id>: { type: 'stdio', command, args, env? } } }`), con
> los servers managed de Axiom (`sdd-mcp-server`, `spec-mcp-broker`).
> Esto es una ASUNCIÓN DOCUMENTADA y override-able para Visual Studio
> 2022 17.14+/2026: el schema no fue re-verificado de forma
> independiente contra Visual Studio real (solo está documentado que VS
> lee MCP config desde ubicaciones a nivel solución). Si tu build de
> Visual Studio no lee `.vs/mcp.json`, agregá esos servers manualmente a
> tu configuración de MCP de Visual Studio.

## Validation

From `Axiom`:
- `npm run build` — passed (tsc -b, no errors).
- `npx vitest run packages/adapters/visual-studio-2026 packages/adapters/antigravity packages/adapters/codex apps/cli/tests/workspace-adapters.test.ts apps/cli/tests/e2e/adapters.e2e.test.ts` —
  5 test files, 59 tests, all passed. No test files required edits.

## Result

All 4 stale-text targets fixed and verified against a real build + the
full relevant test surface. Working tree left uncommitted per
instructions (no git add/commit/push performed).

## General spec integration

No stable, product-level knowledge to integrate: this is a text-only
consistency fix within the same batch's already-integrated behavior
(the MCP-parity behavior itself was already the subject of INC-3's own
general-spec integration, if any). Nothing new to add to
`general-spec.md`.
