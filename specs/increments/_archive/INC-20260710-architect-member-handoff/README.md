# Increment: architect -> member handoff (mcp-manifest.yaml / toolchain-catalog.yaml gap)

Status: closed
Date: 2026-07-10

## Goal

Fix a broken producer/consumer contract between the ARCHITECT flow
(`axiom workspace setup`) and the MEMBER flow (`axiom member install`):
the architect never scaffolded the two committed declaration files
(`axiom.config/mcp-manifest.yaml`, `axiom.config/toolchain-catalog.yaml`)
that the member flow reads to materialize per-user native MCP config and
report the toolchain catalog. Also verify/fix the MCP launch command
resolution and its cosmetic rendering in the member-install summary.

## Context

Found live during an autopilot cross-increment smoke test: an architect
ran `axiom workspace setup`, then a simulated fresh member clone ran
`axiom member install --member alice`, which printed `config MCP nativo:
0 archivo(s)` plus a warning `No se pudo cargar mcp-manifest.yaml … No se
encontró mcp-manifest.yaml`. Root cause: `runWorkspaceSetup`
(`workspace-setup.ts`) writes `.axiom/mcp.yml` (the GENERATED canonical
project config) but never writes `axiom.config/mcp-manifest.yaml` nor
`axiom.config/toolchain-catalog.yaml` — the DECLARED, committed files
that `axiom member install` (`member-install.ts`) actually reads (via
`loadMcpManifest`, `apps/cli/src/commands/mcp.ts`) to derive native MCP
server entries for each member's machine. The `Axiom` product repo only
had `axiom.config/mcp-manifest.yaml` because a previous increment hand-
seeded it; there was no generation path for a brand-new project.

A second, smaller issue was flagged for verification: the member-install
summary line rendered `launch command = "axiom "` (trailing space before
the closing quote) when `baseArgs` is empty. Root cause confirmed: the
line applied `.trimEnd()` to the WHOLE summary string, which ends in
`".` (closing quote + period), not in the space itself — so `trimEnd()`
never touched the space that sits between `axiom` and the empty
`baseArgs.join(' ')`. The underlying `resolveMcpLaunchCommand`/
`isAxiomOnPath` resolver logic itself was already correct (confirmed by
existing injected-override unit tests); this was purely a rendering bug,
now additionally covered by real-PATH-manipulation tests (no injected
overrides) proving the actual production fallback path.

## Scope

1. New `apps/cli/src/commands/workspace-config-scaffold.ts`:
   `scaffoldMcpManifestIfMissing`, `scaffoldToolchainCatalogIfMissing`,
   and the combined `scaffoldArchitectDeclarations` wrapper. No-clobber
   per file (never overwrites an existing file — including a hand-seeded
   one like `Axiom`'s own `mcp-manifest.yaml`); best-effort (I/O failures
   degrade to a warning, never throw).
2. `runWorkspaceSetup` (`workspace-setup.ts`) calls
   `scaffoldArchitectDeclarations(control.path)` as a new best-effort
   step (2c, right after the existing `profiles.yaml` scaffold step),
   appending its `filesCreated`/`warnings`.
3. New granular, re-runnable CLI command `axiom workspace
   config-scaffold` (`workspace.ts`, `runWorkspaceConfigScaffold`),
   following the exact pattern of the other 5 granular subcommands from
   INC-20260710-workspace-command-parity (`spec-base`/`adapters`/
   `skills`/`rules`/`mcp-config`) — resolves the existing project via
   `resolveExistingProject` and re-invokes `scaffoldArchitectDeclarations`
   on its control repo path. Lets an operator repair a pre-existing
   install (bootstrapped before this increment) without re-running full
   setup.
4. Cosmetic fix in `member-install.ts`: the summary line now joins
   `command` + non-empty `baseArgs` tokens with a single space, instead
   of relying on a whole-line `.trimEnd()` that never reached the
   trailing space.
5. Tests: new unit tests for the scaffold module (happy path, no-clobber,
   best-effort-on-I/O-failure); new scenario in `workspace-setup.test.ts`
   (scaffold-on-setup, no-clobber against a pre-seeded file, idempotent
   re-run); new scenario in `workspace-command.test.ts` for the granular
   command; new real-PATH-manipulation tests in `workspace-mcp.test.ts`
   for `isAxiomOnPath`/`resolveMcpLaunchCommand` (no injected overrides);
   new end-to-end handoff scenarios in `member-install.test.ts` running
   the REAL `runWorkspaceSetup` -> simulated clone (no `.axiom-state/`)
   -> REAL `runMemberInstall`, plus a no-clobber scenario against a
   pre-seeded control repo (mirroring the `Axiom` product repo's own
   situation).

## Non-goals

- No change to the underlying `resolveMcpLaunchCommand`/`isAxiomOnPath`
  resolution logic itself — it was already correct; this increment only
  added real-PATH tests to prove it end-to-end and fixed the unrelated
  cosmetic rendering bug in the summary line.
- No change to `.axiom/mcp.yml` (the GENERATED canonical project config)
  or its generation path (`writeWorkspaceMcpConfig`) — unaffected, still
  the architect-side artifact it always was.
- No enrichment of the minimal `mcp-manifest.yaml`/`toolchain-catalog.yaml`
  shapes scaffolded here beyond what the existing tolerant loaders
  (`mcp.ts`'s `isRawMcpEntryLike`, `toolchain.ts`'s
  `loadToolchainCatalogFile`) already accept — same minimal shape as the
  `Axiom` product repo's own hand-seeded files.
- No `--home-dir` flag added to `axiom workspace config-scaffold` (same
  convention as the other 5 granular commands from INC-20260710-
  workspace-command-parity: `homeDirOverride` stays a programmatic-only
  param for tests/advanced callers).

## Acceptance criteria

- [x] `axiom workspace setup` scaffolds `axiom.config/mcp-manifest.yaml`
      and `axiom.config/toolchain-catalog.yaml` in the control repo when
      they don't already exist.
- [x] Neither file is ever clobbered if already present (verified both
      against a synthetic pre-seeded fixture and directly against the
      real `Axiom` product repo's own files).
- [x] `axiom member install` on a fresh clone of an `axiom workspace
      setup` output now materializes >=1 native MCP config file with a
      resolvable launch command, and no longer warns "no se encontró
      mcp-manifest.yaml".
- [x] The launch-command cosmetic bug (`"axiom "` trailing space) is
      fixed; the resolver's real (non-mocked) PATH-based fallback to
      `node <abs cli path>` is proven by dedicated tests.
- [x] New granular `axiom workspace config-scaffold` command exists,
      re-runnable, no-clobber.
- [x] `npm run build` exits 0.
- [x] Live two-actor e2e proof in scratch temp dirs with sandboxed
      HOME/USERPROFILE and PATH stripped of `axiom` for the member step
      (see Validation).
- [x] Full `npm test` stays green with zero regressions.

## Open questions

None blocking.

## Assumptions

- The default catalog seeded by `scaffoldToolchainCatalogIfMissing`
  reuses the exact same set/shape already established by
  INC-20260710-schema-reconciliation (`serena`/`codegraph`/`graphify`/
  `engram`/`context7`/`rtk`/`caveman`/`autoskills`, all `mvp: false`) —
  no new tools introduced, no new schema fields.
- Folding the new scaffold into a NEW granular command
  (`config-scaffold`) rather than extending `mcp-config` keeps each
  granular command's responsibility narrow and consistent with the
  existing split (`mcp-config` owns `.axiom/mcp.yml` + native adapter
  configs; this new command owns the two committed DECLARATION files
  that `mcp-config`'s own generation does not depend on).

## Implementation notes

- `apps/cli/src/commands/workspace-config-scaffold.ts` (NEW):
  `scaffoldMcpManifestIfMissing`, `scaffoldToolchainCatalogIfMissing`,
  `scaffoldArchitectDeclarations`, `MCP_MANIFEST_YAML_REL`,
  `TOOLCHAIN_CATALOG_YAML_REL`. Same atomic-write (tmp + rename) pattern
  as `scaffoldProfilesYamlIfMissing` in `workspace-setup.ts`.
- `apps/cli/src/commands/workspace-setup.ts`: imports
  `scaffoldArchitectDeclarations`; new best-effort step 2c right after
  the `profiles.yaml` scaffold, appending to `filesCreated`/`warnings`.
- `apps/cli/src/commands/workspace.ts`: new exported
  `runWorkspaceConfigScaffold` + new `config-scaffold` subcommand
  registered in `registerWorkspace`; header comment updated (6 granular
  subcommands now, was 5).
- `apps/cli/src/commands/member-install.ts`: the MCP summary line now
  computes `launchCommandDisplay` by joining `command` + non-empty
  `baseArgs` tokens with `' '`, replacing the previous whole-line
  `.trimEnd()` that never reached the actual trailing space.
- Tests (all new or extended, no existing test behavior changed):
  - `apps/cli/tests/workspace-config-scaffold.test.ts` (NEW, 7 tests).
  - `apps/cli/tests/workspace-setup.test.ts`: new "Scenario (m)" (3
    tests: scaffold-on-setup, no-clobber against a pre-seeded file,
    idempotent re-run).
  - `apps/cli/tests/workspace-command.test.ts`: new `(h)
    runWorkspaceConfigScaffold` describe block (3 tests).
  - `apps/cli/tests/workspace-mcp.test.ts`: new `isAxiomOnPath`/
    `resolveMcpLaunchCommand` real-PATH-manipulation describe blocks (4
    tests, no injected overrides — manipulate `process.env.PATH`/`Path`
    directly, save/restore around each test).
  - `apps/cli/tests/member-install.test.ts`: new "Handoff end-to-end"
    describe block (2 tests: full architect-setup -> clone -> member-
    install handoff with real launch-command resolution + idempotent
    re-run; no-clobber against a pre-seeded control repo mirroring the
    `Axiom` product repo's own situation).

## Validation

- `cd "C:/repos/Axiom Workspace/Axiom" && npm run build` -> exit 0
  (`tsc -b`, clean).
- Targeted test run: `workspace-config-scaffold.test.ts` (7),
  `workspace-mcp.test.ts` (26), `workspace-setup.test.ts` (36),
  `workspace-command.test.ts` (15), `member-install.test.ts` (8) — all
  pass, 92 tests, 0 failures.
- Full `npm test`: **212 files / 2268 tests, all pass** (baseline before
  this increment: 211 files / 2249 tests — this increment added exactly
  1 new test file + 19 new tests across 4 existing files: +7 scaffold
  module, +4 real-PATH resolver, +3 setup scenario, +3 granular command,
  +2 handoff e2e; zero regressions, zero failures).
- LIVE two-actor e2e proof, scratch temp dirs (session scratchpad,
  removed afterwards), sandboxed `HOME`/`USERPROFILE` for the architect
  step and `--home-dir` for the member step (real `~/.axiom` never
  touched):
  - **(a) ARCHITECT**: `node dist/index.js workspace setup --name
    "Handoff Demo" --control-path <tmp>.sdd --spec-path <tmp>.spec
    --adapters claude-code --profile builder --overlay local-only`
    (with `HOME`/`USERPROFILE` pointed at a scratch home) -> exit 0,
    `Registro: OK`. Confirmed BOTH files now exist in the control repo:
    `axiom.config/mcp-manifest.yaml` (schemaVersion 1, `sdd`/`spec`,
    both `projectBinding: required`) and `axiom.config/toolchain-
    catalog.yaml` (8 tools, all `mvp: false`) — full contents pasted
    below.
  - **(b) MEMBER**: copied the control+spec repos to a fresh location
    EXCLUDING `.axiom-state/` (simulating `git clone`; confirmed absent
    with a direct filesystem check), then ran `node dist/index.js member
    install --member alice --home-dir <sandbox>` with `PATH`/`Path` set
    to an empty scratch directory (confirmed no `axiom` binary present)
    -> exit 0. BEFORE this fix the repro showed `config MCP nativo: 0
    archivo(s)` + a "no se encontró mcp-manifest.yaml" warning; AFTER:
    `config MCP nativo: 10 archivo(s), launch command = "node
    C:\repos\Axiom Workspace\Axiom\apps\cli\dist\index.js"` — no
    trailing-space bug, no missing-manifest warning. The materialized
    `.mcp.json` (pasted below) shows `"command": "node"` with the
    absolute CLI entry path as the first arg — resolvable regardless of
    the member's PATH. Ran the SAME command a second time: identical
    `sha1sum` of `.mcp.json` before/after, `alice` reported as "ya
    estaba registrado" (idempotent, exit 0 both times).
  - **(c)** Called `scaffoldArchitectDeclarations('.')` directly (via
    the compiled `dist/commands/workspace-config-scaffold.js`) against
    the REAL `Axiom` product repo root: `{ filesCreated: [], warnings:
    [] }` — confirmed via `sha1sum` that both
    `axiom.config/mcp-manifest.yaml` and `axiom.config/toolchain-
    catalog.yaml` are byte-for-byte unchanged before/after (checksums
    identical: `a14cdec0...` and `708c6de1...` respectively).

  `mcp-manifest.yaml` (architect step, freshly scaffolded):
  ```yaml
  # MCP manifest de Axiom (scaffoldeado por "axiom workspace setup" —
  # INC-20260710-architect-member-handoff).
  #
  # Declara los MCP servers sdd/spec bound al proyecto activo.
  # projectBinding: required es obligatorio para ambos ids
  # (mcp-bindings-coherence, @axiom/doctor's TC-007). "axiom member
  # install" lee este archivo para materializar el config MCP nativo en
  # la máquina de cada miembro del equipo — nunca lo edites a mano salvo
  # para agregar/quitar MCPs declarados.
  schemaVersion: 1
  mcps:
    - id: sdd
      server: sdd-mcp-server
      projectBinding: required
    - id: spec
      server: spec-mcp-broker
      projectBinding: required
  ```

  `.mcp.json` (member step, materialized on a machine without `axiom`
  on PATH):
  ```json
  {
    "mcpServers": {
      "sdd-mcp-server": {
        "command": "node",
        "args": [
          "C:\\repos\\Axiom Workspace\\Axiom\\apps\\cli\\dist\\index.js",
          "mcp", "serve", "--kind", "sdd", "--project-root", "<member control repo abs path>"
        ]
      },
      "spec-mcp-broker": {
        "command": "node",
        "args": [
          "C:\\repos\\Axiom Workspace\\Axiom\\apps\\cli\\dist\\index.js",
          "mcp", "serve", "--kind", "spec", "--project-root", "<member spec repo abs path>"
        ]
      }
    }
  }
  ```

## Result

Closed. The architect -> member handoff gap is fixed at the producer:
`axiom workspace setup` now scaffolds both committed declaration files
(no-clobber, best-effort) that `axiom member install` depends on. A new
granular `axiom workspace config-scaffold` command lets an operator
repair a pre-existing install without a full re-setup. The MCP launch-
command resolver itself was already correct; a real-PATH (no mocks) test
suite now proves the production fallback path end-to-end, and the
unrelated cosmetic trailing-space bug in the install summary is fixed.
Live two-actor proof confirms the exact repro (0 native MCP files ->
>=1, with a resolvable `node <abs path>` launch command) is closed, that
re-running either side is idempotent, and that the `Axiom` product
repo's own hand-seeded `mcp-manifest.yaml`/`toolchain-catalog.yaml` are
never touched. Full suite green: 212 files / 2268 tests, zero
regressions.

## General spec integration

`Axiom.Spec/general-spec.md` does not exist in this repo's structure
(same situation already noted by INC-20260710-dynamic-team-roles and
INC-20260710-workspace-command-parity); canonical stable knowledge
instead lives in `specs/05_Interfaces_Operativas.md` (the CLI surface
reference) — used that file as the equivalent. Integrated: a note next
to the existing `axiom workspace setup`/`axiom member install`
documentation clarifying that `workspace setup` now scaffolds
`axiom.config/mcp-manifest.yaml` + `axiom.config/toolchain-catalog.yaml`
(no-clobber) as part of the architect-side contract that `member
install` depends on, and documenting the new `axiom workspace
config-scaffold` granular repair command alongside the other 5 from
INC-20260710-workspace-command-parity.
