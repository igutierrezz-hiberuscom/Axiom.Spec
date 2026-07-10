# Increment: MCP launch config wiring

Status: closed
Date: 2026-07-08

## Goal

`INC-20260708-mcp-runnable-server` shipped a real, runnable MCP server
(`@axiom/mcp-server` + `axiom mcp serve --kind <sdd|spec> [--project-root
<path>] [--home-dir <path>]`, launcher in `apps/cli/src/commands/
mcp-serve.ts`). But the generated MCP configs (`.axiom/mcp.yml`,
`.opencode/mcp.json`, `.claude/mcp.json`) only DECLARE `sdd-mcp-server` /
`spec-mcp-broker` as string identifiers — they carry no launch command, so
nothing actually starts the server. This increment makes the generated
configs carry a REAL launch command (`command`/`args`) so the declared
entries point at `axiom mcp serve` with the right `--kind` and
`--project-root`.

## Context

Read first: `apps/cli/src/commands/mcp-serve.ts` (confirms the exact CLI
surface: `axiom mcp serve --kind <sdd|spec> [--project-root <path>]
[--home-dir <path>]`), `packages/user-workspace/src/mcp-config.ts`
(`McpServerEntry`/`McpProjectConfig`/`validateMcpProjectConfig`),
`apps/cli/src/commands/workspace-mcp.ts` (`buildWorkspaceMcpServers` +
`writeWorkspaceMcpConfig`), and both adapter generators
(`packages/adapters/opencode/src/mcp-json.ts`,
`packages/adapters/claude-code/src/mcp-json.ts`).

Today `buildWorkspaceMcpServers` emits `sdd-mcp-server`
(`type:'axiom'`, `scope:'repo'`, `targetRepo:<control roleKey>`,
`enabled:true`) and `spec-mcp-broker` (same shape, `targetRepo:<spec
roleKey>`), with NO field describing how to actually start the process
behind those ids. `writeWorkspaceMcpConfig` writes those entries verbatim
to `.axiom/mcp.yml` and projects them (stripped down to
`id`/`type`/`scope`/`targetRepo`) into `.opencode/mcp.json` /
`.claude/mcp.json` via `generateOpencodeMcpJson`/`generateClaudeCodeMcpJson`.

## Scope

1. `packages/user-workspace/src/mcp-config.ts`: extend `McpServerEntry`
   with OPTIONAL launch fields — `command?: string`, `args?: readonly
   string[]`, `env?: Readonly<Record<string,string>>`. Purely additive;
   `validateMcpProjectConfig`'s shape guard (`isMcpServerEntryLike`) only
   validates them when present (type-shape only, no new required field or
   semantic rule).
2. `packages/adapters/opencode/src/mcp-json.ts` and
   `packages/adapters/claude-code/src/mcp-json.ts`: extend
   `McpServerEntryInput` identically with the same optional
   `command`/`args`/`env`, spread-conditionally into the emitted JSON per
   server entry (same style as the existing optional `targetRepo`). Atomic
   write + `enabled`-filter behavior unchanged.
3. `apps/cli/src/commands/workspace-mcp.ts`:
   - `buildWorkspaceMcpServers` gains an extra parameter carrying the
     absolute control-repo path and absolute spec-repo path (both already
     resolved by the caller, `runWorkspaceSetup`), and populates
     `command: 'axiom'` + `args: ['mcp', 'serve', '--kind', 'sdd'|'spec',
     '--project-root', <absolute path>]` on the two emitted entries.
   - `writeWorkspaceMcpConfig` passes `command`/`args`/`env` through to
     both `.axiom/mcp.yml` (already dumps full entries) and the adapter
     projection input (`inputServers` mapping).
   - The degenerate control===spec roleKey collapse behavior (emit only
     `sdd-mcp-server`) is unchanged.
   - `runWorkspaceSetup`'s call site updated to pass the two absolute repo
     paths it already has (`control.path`, `specRepo.path`).
4. Backward compatibility: entries without launch fields (e.g. built by
   hand, by older code, or by any caller not going through
   `buildWorkspaceMcpServers`) continue to validate and to project without
   a `command`/`args`/`env` key in the output JSON, byte-for-byte
   identical to pre-increment output.

## Non-goals

- Mapping Axiom's own `.opencode/mcp.json` / `.claude/mcp.json` document
  shape (`{schemaVersion, projectId, servers:[...]}`) onto each tool's
  actual NATIVE MCP-launch config schema (e.g. Claude Code's real
  `mcpServers` object, or opencode's real MCP config surface) is
  explicitly OUT of scope. This increment makes `command`/`args`
  authoritative and present in Axiom's own generated documents so
  downstream tooling/tests can consume them; translating to each
  ecosystem's exact native transport schema is a separate follow-up that
  needs each adapter's verified transport spec (not invented here).
- No new npm dependencies.
- No changes to `00`–`08` canonical spec docs.
- No changes to `@axiom/mcp-server` / `mcp-serve.ts` themselves (the
  runnable server and its CLI surface are already correct and unchanged
  here).

## Acceptance criteria

- [x] `McpServerEntry` (`packages/user-workspace/src/mcp-config.ts`) has
      optional `command`/`args`/`env`; `validateMcpProjectConfig` still
      passes for entries with AND without these fields.
- [x] `McpServerEntryInput` in both adapter generators has the same
      optional fields, emitted into the JSON output when present, omitted
      when absent.
- [x] `buildWorkspaceMcpServers` emits `command:'axiom'` and the correct
      `args` (`mcp serve --kind sdd|spec --project-root <absolute path>`)
      for both `sdd-mcp-server` and `spec-mcp-broker`.
- [x] `.axiom/mcp.yml` and the adapter `.opencode/mcp.json` /
      `.claude/mcp.json` produced by `writeWorkspaceMcpConfig` /
      `runWorkspaceSetup` contain the launch fields.
- [x] All new + existing targeted tests pass: `packages/adapters/opencode/
      tests/mcp-json.test.ts`, `packages/adapters/claude-code/tests/
      mcp-json.test.ts`, `apps/cli/tests/workspace-mcp.test.ts`,
      `packages/user-workspace/tests/mcp-config.test.ts`.
- [x] `npm run build` (tsc -b) clean.
- [x] `npx vitest run apps/cli packages/adapters packages/user-workspace`
      passes (pre-existing `packages/skills/tests/catalog.test.ts` dogfood
      failure and known pre-existing dogfood/integration failures are
      out of scope; any other failure is classified).

## Open questions

None blocking — resolved under "Assumptions".

## Assumptions

- **`command: 'axiom'`**: assumes the Axiom CLI is installed globally and
  resolvable on `PATH` as `axiom` wherever the generated config is
  consumed. This is the same assumption every other Axiom-generated
  config in this repo makes implicitly (none of them currently embed an
  absolute path to the CLI binary). If `axiom` is not globally installed,
  the operator must adjust the generated `command` by hand, or a future
  increment could add an env-var override; not solved here (would be
  speculative without a concrete second use case).
- **Absolute paths for `--project-root`**: `buildWorkspaceMcpServers`
  already receives `WorkspaceSetupResult['repos']` (each entry has a
  resolved `path`) and `allRepoSpecs` (each has a `path` too, pre-resolution).
  The absolute control/spec repo paths are threaded in as two new
  parameters (`controlRepoAbsolutePath`, `specRepoAbsolutePath`) rather
  than re-deriving them inside `buildWorkspaceMcpServers`, since the
  caller (`runWorkspaceSetup`) already has `control.path`/`specRepo.path`
  resolved to absolute paths (confirmed by reading `workspace-setup.ts`).
- **`env` is currently always omitted**: no launch scenario needs an env
  override yet; the field exists in the type/shape for forward
  compatibility (e.g. a future `--home-dir` override could be passed via
  args instead, so `env` stays unused today, never emitted).
- **`--home-dir` not passed by default**: `resolveMcpServerContext`/
  `runMcpServe` already default `homeDir` to the real `~/.axiom` when
  `--home-dir` is omitted, which is the correct behavior for a real
  operator machine; `buildWorkspaceMcpServers` does not have a
  `homeDirOverride` concept that would need to survive into a spawned
  child process (that override exists purely for this repo's own test
  isolation), so `args` never includes `--home-dir`.
- **Validation of launch fields kept shape-only**: `isMcpServerEntryLike`
  gained shape checks (string / string[] / Record<string,string>, all
  optional) but no NEW semantic rule (e.g. "command must be non-empty
  when args is present") was added to `validateMcpProjectConfig`'s six
  numbered rules — out of scope per the design brief's explicit
  instruction to keep this additive and not invent new semantics.

## Implementation notes

- `packages/user-workspace/src/mcp-config.ts`: `McpServerEntry` gained
  `command?: string`, `args?: readonly string[]`, `env?:
  Readonly<Record<string,string>>`. `isMcpServerEntryLike` gained shape
  guards for the three new optional fields (same pattern as the existing
  `targetRepo` optional-string guard).
- Both adapter `mcp-json.ts` files: `McpServerEntryInput` gained the same
  three optional fields; the emitted document's per-server mapping gained
  `...(s.command !== undefined ? { command: s.command } : {})`,
  `...(s.args !== undefined ? { args: s.args } : {})`,
  `...(s.env !== undefined ? { env: s.env } : {})`, spread AFTER
  `targetRepo`'s existing spread, preserving key order
  `id, type, scope, targetRepo?, command?, args?, env?`.
- `apps/cli/src/commands/workspace-mcp.ts`:
  - `buildWorkspaceMcpServers(repos, allRepoSpecs, launchPaths)` gained a
    third parameter `{ controlRepoAbsolutePath: string;
    specRepoAbsolutePath: string }`. Both emitted entries now set
    `command: 'axiom'` and the corresponding `args` array. The
    control===spec collapse case only needs `controlRepoAbsolutePath`
    (the single emitted `sdd-mcp-server` entry targets the control repo
    path, matching its pre-existing `targetRepo` semantics).
  - `writeWorkspaceMcpConfig`'s `inputServers` mapping gained `command:
    s.command, args: s.args, env: s.env` (adapter generators handle
    `undefined` via their own conditional spread, so passing `undefined`
    through is safe and matches the existing `targetRepo: s.targetRepo`
    line's own style).
  - `runWorkspaceSetup`'s call site updated: `buildWorkspaceMcpServers(
    repoResults, spec.repos, { controlRepoAbsolutePath: control.path,
    specRepoAbsolutePath: specRepo.path })`.
- Existing exported constants (`SDD_MCP_SERVER_ID`, `SPEC_MCP_BROKER_ID`,
  `MCP_PROJECT_CONFIG_RELATIVE_PATH`) untouched.

## Validation

### `npm run build` (tsc -b), from `Axiom/`

```
> axiom-product@0.1.0 build
> tsc -b

(clean exit, no errors)
```

### `npx vitest run apps/cli packages/adapters packages/user-workspace`

```
 Test Files  70 passed (70)
      Tests  634 passed (634)
```

All new assertions (launch-field presence/absence in both adapter
generators' output, `buildWorkspaceMcpServers`' `command`/`args`,
`.axiom/mcp.yml` + `.opencode/mcp.json` launch-field propagation via
`runWorkspaceSetup`, and `validateMcpProjectConfig`/`loadMcpProjectConfig`
backward-compat with and without launch fields) pass. Zero regressions in
the target scope (627 pre-existing + 7 new tests, all green).

### Full-repo `npx vitest run` (extra check, not required by the increment)

```
 Test Files  10 failed | 165 passed (175)
      Tests  10 failed | 1806 passed (1816)
```

Same 10 pre-existing dogfood/environment failures documented in
`INC-20260708-mcp-runnable-server`'s own spec (agents/skills catalog
reading this workspace's own fixtures, model-routing reading the real
`axiom.spec` policy, `resolveProject`'s real-repo scope test,
`runDoctorChecks`'s real-repo test, `audit-trail-sink.test.ts`'s retention
sweep timing, `repair-add-gitignore.test.ts`). Confirmed none touch
`mcp-config.ts`, `workspace-mcp.ts`, `workspace-setup.ts`, or either
adapter's `mcp-json.ts` — zero new regressions from this increment's diff.

## Result

Implemented as scoped. The two workspace-generated MCP entries
(`sdd-mcp-server`, `spec-mcp-broker`) now carry a real, authoritative
launch command (`axiom mcp serve --kind <sdd|spec> --project-root
<absolute path>`) in `.axiom/mcp.yml` and in both adapter-projected
`mcp.json` files. Entries without launch fields (hand-written or
pre-existing) continue to validate and project exactly as before —
confirmed by both the added backward-compat tests and the unchanged
byte-shape of the omitted-field code path (`command !== undefined` guards).
The one explicitly-scoped-out item — mapping onto each adapter's real
NATIVE MCP config schema — is documented as a follow-up, not invented.

## General spec integration

No `general-spec.md`-equivalent stable-knowledge file exists yet in
`Axiom.Spec/specs/` (confirmed absent; the closest analogues are the
numbered `Axiom.Spec/context/architecture/0X-*.md` docs and
`Axiom.Spec/context/integrations/01-capabilities-providers-y-toolchain.md`,
both out of scope to restructure here per the "do not touch 00-08"-style
guardrail carried over from the parent increment). The stable knowledge
this increment introduces — that `sdd-mcp-server`/`spec-mcp-broker`
entries in `.axiom/mcp.yml` and their adapter projections now carry a real
`command`/`args` launch command pointing at `axiom mcp serve`, and that
mapping this onto each adapter's native MCP schema remains an open
follow-up — is recorded here, in `apps/cli/src/commands/workspace-mcp.ts`'s
own header comment, and should be linked from a future increment that
revisits `Axiom.Spec/context/integrations/01-capabilities-providers-y-toolchain.md`.

### Integración canónica realizada (pase de round 4, 2026-07-08)

El pase de integración de round 4 (orquestador) integró el conocimiento
estable de este incremento en los ficheros canónicos `00–08`:

- **`06_Integraciones_y_Capacidades.md`**: la subsección "Server MCP
  ejecutable" documenta que `sdd-mcp-server`/`spec-mcp-broker` generados
  por `runWorkspaceSetup` ahora llevan `command:'axiom'` +
  `args:['mcp','serve','--kind',<sdd|spec>,'--project-root',<path>]` tanto
  en `.axiom/mcp.yml` como en la proyección de adapter; también se
  actualizó la entrada de la subsección "Generación de config MCP durante
  el setup de workspace" para referenciarlo. Se conserva el caveat honesto
  de que el mapeo al schema MCP nativo de cada adapter sigue pendiente.
- **`03_Modelo_Operativo_y_Datos.md`**: `McpServerEntry` documentado con
  los campos opcionales/aditivos `command?`/`args?`/`env?`
  (backward-compatible, validación shape-only); supersede la nota
  previa de "no se inventan campos de transporte".
- **`08_Glosario.md`**: término "Launch config de MCP (`command`/`args`)"
  añadido en la nueva sección de tanda 4.
