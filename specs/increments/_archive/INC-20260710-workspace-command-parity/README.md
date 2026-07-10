# Increment: workspace command parity (P1-6 + "todo lo del wizard TUI por comando")

Status: closed
Date: 2026-07-10

## Goal

Let an operator do, headlessly and by CLI command (no TTY, no
interactive prompts), everything the interactive TUI wizard does to
stand up a multi-repo Axiom workspace (control/SDD + spec + N repos of
functional role) — and let them re-run/repair/reuse just ONE part of an
existing install without re-running the whole setup.

## Context

Audit-confirmed gap: the pure engine `runWorkspaceSetup`
(`Axiom/apps/cli/src/commands/workspace-setup.ts`, input
`WorkspaceSetupSpec`) had exactly ONE caller —
`tui.ts`'s `runNoProjectBootstrap`, which requires an interactive TTY.
There was no headless command: `axiom workspace --help` fell through
to the top-level help (no `workspace` command registered at all in
`index.ts`). Consequence: no way to script a full install, run it in
CI, or re-run/repair just one part of an existing install. Also,
`axiom init --layout installed-multi-repo` only wrote the control
repo's `axiom.yaml` (with a `specification: { path: ../<name>.spec,
... }` reference) and never actually created the spec repo — a silent
gap.

Two ADD-only incremental command groups already existed and are
UNCHANGED by this increment: `axiom repo add` / `axiom adapter add` /
`axiom provider add` / `axiom role add`
(`workspace-incremental.ts`, INC-20260708-incremental-operations) and
`axiom roles register/unregister/assign/unassign`
(`roles.ts`, INC-20260710-dynamic-team-roles). Those cover "add
something NEW to an existing install"; this increment adds the
complementary headless full-setup command plus "re-apply/repair a
PART of an install" granular commands.

## Scope

1. New `axiom workspace` command group
   (`apps/cli/src/commands/workspace.ts`, `registerWorkspace(program)`,
   wired in `index.ts` right after `registerWorkspaceIncremental`).
   Fully non-interactive: every input is a flag; missing required
   flags fail with a clear message (via commander's `requiredOption`
   for `--name`/`--spec-path`, and thrown `Error`s — caught and printed
   to stderr with exit 1 — for flag-value validation).
2. `axiom workspace setup` — headless full setup. Maps flags to a
   `WorkspaceSetupSpec` (via the new pure, exported
   `buildWorkspaceSetupSpecFromFlags`, the headless equivalent of
   `tui.ts`'s `buildWorkspaceSetupSpecFromWizard`) and calls the
   EXISTING `runWorkspaceSetup` engine unchanged. Flags: `--name`
   (required), `--control-path` (default cwd), `--spec-path`
   (required), `--role <name>:<path>` (repeatable — dynamic team
   roles, 1..N, no fixed enum, per Decision D5), `--adapters <csv>`
   (default `opencode`), `--profile`, `--overlay`, `--providers <csv>`,
   `--no-register`, `-y/--yes` (accepted no-op — this command never had
   prompts to skip; kept only for flag-surface parity with `axiom
   init`), `--json`.
3. Granular, re-runnable subcommands for repairing/reusing PARTS of an
   install — each resolves the existing project from cwd via the SAME
   `resolveExistingProject` helper that `repo add`/`adapter add`
   already use (exported from `workspace-incremental.ts` for this
   increment, not duplicated), and re-invokes the corresponding
   already-existing, already no-clobber/deterministic step fn, WITHOUT
   gating on "recently created" (unlike the full engine, which only
   ever touches brand-new repos for skills/rules) — that gate removal
   is precisely the point of a repair/reuse command:
   - `axiom workspace spec-base --spec-path <dir> [--name <p>]` →
     `scaffoldSpecRepoBase`. Does not require the project to be
     registered — only needs the spec repo's path.
   - `axiom workspace adapters [--path <repo>] [--adapters <csv>]` →
     `generateWorkspaceAdapters`. Regenerates adapter *output* only
     (AGENTS.md/skills-lock/etc. per adapter) — never touches
     `workspace.json` (that remains `adapter add`'s job) and never
     touches native MCP config (that is `mcp-config`'s job).
   - `axiom workspace skills [--path <repo>]` → `scaffoldSddSkills` /
     `scaffoldCodeRepoSkills` depending on whether the target repo is
     the control repo or a role repo (spec repos are skipped with a
     warning, matching the engine's own rule).
   - `axiom workspace rules [--path <repo>]` → `scaffoldRules` (control
     gets `common` only; role repos also get `inferRepoLanguages`;
     spec is skipped). Already no-clobber per file.
   - `axiom workspace mcp-config [--path <repo>]` →
     `buildWorkspaceMcpServers` + `writeWorkspaceMcpConfig` +
     `writeWorkspaceNativeMcpConfigs` (same 3 helpers the engine's own
     step 4 uses).
4. Small, low-risk exports added to `workspace-incremental.ts` so the
   new file can reuse rather than duplicate: `resolveExistingProject`,
   `ResolvedExistingProject`/`ResolvedWorkspaceRepo`,
   `locateOrDefaultWorkspaceJsonPath`, `readWorkspaceJson`/
   `WorkspaceJsonRecord`, and a NEW extracted helper
   `buildRepoSpecsFromResolvedProject` (previously an inline block only
   inside `runRepoAdd`; now shared by `runRepoAdd` itself and by the 4
   granular commands above that need the full `WorkspaceRepoSpec[]`
   reconstruction). No behavior change to `runRepoAdd`.
5. `axiom init --layout installed-multi-repo` gap: chose the SMALLER
   correct fix (documented pointer, not silent multi-repo scaffolding
   inside `init`, which would have duplicated `runWorkspaceSetup`'s
   wiring logic for a command that structurally only ever writes ONE
   repo). `registerInit`'s action now prints an explicit note when
   `result.layout === 'installed-multi-repo'`, naming the exact
   pointer command (`axiom workspace setup --name <nombre> --spec-path
   <dir> [--role <name>:<path> ...]`) that performs the real, complete,
   cross-wired multi-repo scaffold.

## Non-goals

- No change to `axiom repo add`/`adapter add`/`provider add`/`role
  add`'s behavior or CLI surface (only a handful of their private
  helpers gained an `export` keyword for reuse; `runRepoAdd`'s only
  change is calling the extracted `buildRepoSpecsFromResolvedProject`
  instead of an inline duplicate).
- No `--home-dir` CLI flag added to any command (existing convention:
  `homeDirOverride` is a programmatic-only param for tests; real CLI
  invocations always resolve `~/.axiom` via `os.homedir()`). The live
  e2e proof below uses a `USERPROFILE`/`HOME` env override instead, to
  avoid touching the real user home during validation.
- No `remove`-style operations (`workspace adapters` never removes an
  adapter from `workspace.json`; there is still no `repo remove`) —
  out of scope, consistent with INC-20260708-incremental-operations'
  own deferred scope.
- `axiom init` itself is NOT changed to create a second repo — see
  Scope item 5's rationale.
- No new architecture: every granular command is a thin,
  commander-wrapped CLI surface over an EXISTING, already-tested step
  fn. No new scaffolding logic was written.

## Acceptance criteria

- [x] `axiom workspace --help` lists `setup`, `spec-base`, `adapters`,
      `skills`, `rules`, `mcp-config`.
- [x] `axiom workspace setup` runs headlessly (no TTY needed) and
      produces the same cross-wired multi-repo result as the TUI
      wizard for an equivalent input.
- [x] Each granular subcommand reuses the existing step fn/engine
      helper (no scaffolding logic duplicated) and is safe to re-run
      (idempotent or no-clobber, per each step fn's own existing
      contract).
- [x] `axiom init --layout installed-multi-repo` prints an explicit,
      exact pointer to `axiom workspace setup` instead of silently
      leaving the spec-repo gap undocumented.
- [x] `npm run build` exits 0.
- [x] Live headless e2e proof in a scratch temp dir (spec repo + 2 role
      repos + 2 adapters): the 3 repos exist and are cross-wired (each
      `axiom.yaml` references the others), `axiom topology validate`
      passes, the spec repo has its skeleton, and native MCP config
      files were written — plus ONE granular re-run (`workspace
      adapters`) proven idempotent (identical file content before/
      after).
- [x] New tests in `apps/cli/tests/workspace-command.test.ts` covering
      `workspace setup` (headless, temp dir) + all 5 granular
      subcommands.
- [x] Full `npm test` stays green, zero regressions.

## Open questions

None blocking.

## Assumptions

- `axiom workspace adapters` is deliberately narrower than `axiom
  adapter add`: it never persists to `workspace.json#adapters` (that
  remains the ADD command's job) — it only re-generates the adapter
  *output* for the adapters it's told about (explicit `--adapters`, or
  whatever is already enabled). This keeps "add something new" and
  "repair/regenerate something existing" as two distinct,
  non-overlapping verbs, consistent with the brief's own framing
  ("comandos para... arreglar instalaciones o reutilizar solo partes
  de instalación").
- The granular commands' `--path` flag matches by resolved absolute
  path against the project's known repos (from the registry); if it
  doesn't match any known repo, the command fails with exit 1 and a
  clear message listing the known roleKeys, rather than silently
  operating on an arbitrary unrelated directory.
- `-y/--yes` on `workspace setup` is accepted but intentionally a
  no-op: this command was designed to never prompt in the first place
  (that is the entire point of "headless"), so there is nothing for
  `--yes` to skip. It exists purely so a script that uniformly passes
  `--yes` to every Axiom bootstrap command (mirroring `axiom init
  --yes`) does not need a special case for `workspace setup`.
- `--role <name>:<path>` splits on the FIRST `:` in the raw flag value
  (not the last), which is what makes it safe with Windows absolute
  paths that themselves contain a drive-letter colon (e.g.
  `backend:C:\repos\foo-backend`) — the role name never contains a
  colon (validated against `[a-z0-9][a-z0-9-]*`), so the first colon
  in the string is always the intended separator.

## Implementation notes

- `apps/cli/src/commands/workspace.ts` (NEW): `registerWorkspace`,
  `buildWorkspaceSetupSpecFromFlags` (+ `WorkspaceSetupFlags`),
  `runWorkspaceSpecBase`, `runWorkspaceAdapters`, `runWorkspaceSkills`,
  `runWorkspaceRules`, `runWorkspaceMcpConfig`, plus private CLI-flag
  parsing/validation helpers (`parseCsv`, `assertValidEnum(List)`,
  `parseRoleFlag`, `collect`).
- `apps/cli/src/index.ts`: imports and calls `registerWorkspace(program)`
  right after `registerWorkspaceIncremental(program)`.
- `apps/cli/src/commands/workspace-incremental.ts`: exported
  `resolveExistingProject`, `ResolvedExistingProject`,
  `ResolvedWorkspaceRepo`, `locateOrDefaultWorkspaceJsonPath`,
  `readWorkspaceJson`, `WorkspaceJsonRecord` (all previously
  module-private); added the new exported
  `buildRepoSpecsFromResolvedProject` helper and switched `runRepoAdd`
  to call it instead of its previous inline duplicate block (dead
  `existingControl`/`existingSpec` locals removed as part of the same
  edit — they were computed but never read).
- `apps/cli/src/commands/init.ts`: `registerInit`'s action prints an
  explicit pointer note (naming the exact `axiom workspace setup`
  command) when `result.layout === 'installed-multi-repo'`.
- `apps/cli/tests/workspace-command.test.ts` (NEW): 12 tests across 7
  `describe` blocks — `buildWorkspaceSetupSpecFromFlags` (2), the
  headless `workspace setup` integration (1), `runWorkspaceAdapters`
  (4, including idempotency + `--path` + the not-registered error
  path), `runWorkspaceSkills` (2), `runWorkspaceRules` (1),
  `runWorkspaceMcpConfig` (1), `runWorkspaceSpecBase` (1).

## Validation

- `cd Axiom && npm run build` → exit 0 (`tsc -b`, clean, no errors).
- `npx vitest run apps/cli/tests/workspace-command.test.ts` → 1 file,
  12 tests, all pass.
- Full `npm test` (full `vitest run`) → **208 files, 2221 tests, all
  pass** (baseline before this increment, confirmed by a dedicated
  pre-change run: 208 files / 2209 tests, all pass — this increment
  added 1 new test file / 12 new tests, zero regressions, zero
  failures).
- LIVE headless e2e proof, run twice in independent scratch temp dirs
  (session scratchpad, both removed afterwards), with a
  `USERPROFILE`/`HOME` env override pointed at a scratch home dir (so
  the real `~/.axiom` was never touched):
  - `axiom workspace setup --name "Demo App" --control-path <tmp>.sdd
    --spec-path <tmp>.spec --role backend:<tmp>.backend --role
    frontend:<tmp>.frontend --adapters opencode,claude-code --profile
    builder --overlay local-only` → exit 0, `Registro: OK`, all 4 repos
    reported `(nuevo)`.
  - Inspected all 4 generated `axiom.yaml` files directly: `sdd`'s
    `paths` references `spec`/`backend`/`frontend`;
    `spec`'s references `control`/`backend`/`frontend`; each role
    repo's references `control`/`specification`/`product: { path: . }`
    — full bidirectional cross-wiring confirmed.
  - `axiom topology validate --path <tmp>.sdd` → `✓ Topología válida
    (mode: multi-repo).` exit 0.
  - Spec repo skeleton confirmed on disk (`specs/README.md`,
    `specs/00_Resumen_Ejecutivo.md` .. `08_Glosario.md`,
    `context/TECHNICAL_CONTEXT.md`, etc.).
  - Native MCP config confirmed written for the `claude-code` adapter
    (`.mcp.json`) in all 4 repos, plus the canonical `.axiom/mcp.yml`
    in the control repo (2 servers: `sdd-mcp-server`,
    `spec-mcp-broker`).
  - Idempotent granular re-run: `axiom workspace adapters` (no flags —
    defaults to the enabled adapters from `workspace.json`, all repos)
    run once, `sha1sum` of `demo-app.backend/.mcp.json` and
    `opencode.json` captured, run a SECOND time — identical checksums,
    exit 0 both times.
  - Also exercised (not required by the acceptance criteria, but
    verified live for completeness): `workspace skills` (whole
    project — correctly skipped the spec repo with a warning),
    `workspace rules --path <backend>` (correctly reported the
    already-existing `common.md` as no-clobbered on this second
    exposure), `workspace mcp-config` (regenerated both artifacts),
    `workspace spec-base --spec-path <tmp>.spec` (re-run — every file
    reported as already-existing, no-clobbered).
  - The first scratch-dir attempt hit an unrelated environment
    artifact: the session scratchpad's OWN path used a Windows 8.3
    short alias (`IGUTIE~1`) that Windows silently canonicalizes to the
    long form (`igutierrezz`) on an actual `chdir`, but not on a plain
    `path.resolve()` of a literal string — causing a benign registry
    path-string mismatch between the literal `--control-path` value
    used at `setup` time and `process.cwd()` at re-run time. Re-ran
    with the canonical long-form path throughout and it worked
    cleanly; this is a pre-existing characteristic of
    `findByRepoPathV2`'s exact-string path comparison shared by EVERY
    command that resolves an existing project (`repo add`/`adapter
    add`/`provider add` included) — not a regression introduced by
    this increment, and not fixed here (out of scope; the underlying
    helper is unchanged).

## Result

Closed. `axiom workspace` now exposes, by command, the full headless
equivalent of everything the interactive TUI wizard does
(`setup`), plus 5 granular, re-runnable subcommands
(`spec-base`/`adapters`/`skills`/`rules`/`mcp-config`) for repairing or
reusing just one part of an existing install — directly satisfying the
user's explicit requirement ("todo lo que hace el wizard con la TUI
tenga comandos por debajo... para cuando hay que arreglar
instalaciones o reutilizar solo partes de instalación tengamos todo en
comandos"). Every subcommand is command-surface parity over the
EXISTING, already-tested engine and step functions — no new
scaffolding logic, no new architecture. The `axiom init
--layout installed-multi-repo` gap is closed with an explicit,
actionable pointer rather than silent under-scaffolding. Live e2e
proof covers the full 4-repo/2-adapter setup plus one proven-idempotent
granular re-run; automated tests cover the flag-mapping function and
all 5 granular commands; the full suite is green with zero
regressions.

## General spec integration

`Axiom.Spec/general-spec.md` does not exist in this repo's structure
(same situation already noted by INC-20260710-dynamic-team-roles);
canonical stable knowledge instead lives in
`specs/05_Interfaces_Operativas.md` (the CLI surface reference) — used
that file as the equivalent. Integrated: a new subsection documenting
the `axiom workspace` command group (headless full setup + the 5
granular repair/reuse subcommands), cross-referencing the existing
`axiom repo add`/`adapter add`/`provider add`/`role add` ADD-only
commands and clarifying the deliberate split of responsibility between
"add something new" (those commands, plus `roles register`) and
"repair/reuse a part of what already exists" (this increment's
`workspace` subcommands). Also noted, next to the existing `axiom
init` documentation, that `--layout installed-multi-repo` only
scaffolds the single invoking repo and now points explicitly at
`axiom workspace setup` for the real multi-repo scaffold.
