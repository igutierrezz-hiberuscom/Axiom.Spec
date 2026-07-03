# Increment: External integrations as optional plugins — reconcile

Status: closed
Date: 2026-07-02

## Goal

Verify addendum section 10's requirement — Azure DevOps/Jira/Confluence
must be **optional plugins/capabilities**, never part of mandatory core;
`Axiom Core` must function fully without them — against the Azure DevOps
plugin already present in `Axiom/apps/cli/src/commands/
app-plugins-azure-devops.ts` and `app-plugins.ts`. Also verify addendum
section 11's `externalRef`/`externalRefs` model (landed by INC-06/INC-11)
is the actual wiring the plugin uses, or confirm/flag a gap if it uses a
disconnected mechanism instead. This is roadmap INC-16, Phase E.

## Context

The roadmap (`Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-
roadmap/README.md`, INC-16 entry) hypothesized the Azure DevOps plugin is
already ahead of the new docs' description of external integrations as a
"future addition," and framed this increment as verification, not a build:
a migration-engineer audits the existing plugin against addendum §10/§11,
then a validator-reviewer confirms — no implementer subagent in the
planned sequence.

`general-spec.md`'s "MCP tool registration" section already documents a
closely related precedent: 16-17 MCP capability ids are additive,
provider-agnostic, and layered without new mandatory dependencies. This
increment checks whether the same "additive, non-mandatory" posture holds
for the CLI-level Azure DevOps plugin.

## Scope

- Read `app-plugins.ts`, `app-plugins-azure-devops.ts`, `app-api.ts`,
  `app.ts`, and `docs/0030-operator-app-plugins-and-external-bridge.md` in
  full.
- Confirm whether any Axiom Core command (CLI registration in
  `apps/cli/src/index.ts`, `@axiom/doctor`, `@axiom/mcp-tools`,
  `@axiom/workflow`) has a hard dependency on the plugin being
  present/configured.
- Confirm the plugin's actual relationship (or lack thereof) to
  `externalRefs` (`@axiom/workflow`'s `artifact-store.ts`, landed by
  INC-06/INC-11).
- Confirm whether the plugin can be fully absent/disabled without breaking
  any core command.
- Decide whether this increment needs any code change, or closes as a pure
  verification pass.

## Non-goals

- No real Azure DevOps API integration (creating actual work items against
  a live Azure DevOps project). The existing plugin is declarative-only by
  design (GATE 0025/0030) and this increment does not change that.
- No new plugin system, no Jira/Confluence plugin scaffolding.
- No implementation of the `axiom external-sync azure-devops *` commands
  the plugin's `command` fields reference (confirmed as a pre-existing,
  separate, larger gap — see Findings; out of scope for this
  verification-only increment).

## Acceptance criteria

- [x] Confirmed what the existing plugin actually does today (read in
      full, not inferred from filenames).
- [x] Confirmed whether Axiom Core has any hard dependency on the plugin
      (import-graph check from `apps/cli/src/index.ts` and every
      structural package).
- [x] Confirmed the plugin's actual relationship to `externalRefs`
      (wired, disconnected, or not applicable) and made an explicit
      in-scope/out-of-scope call on any gap found.
- [x] Confirmed the plugin can be fully absent without breaking any core
      command.
- [x] Explicit, honest closure decision: verification-only vs. needs a
      fix vs. needs a follow-up increment.
- [x] Recorded in `Axiom.Spec`; no unrelated files modified.

## Open questions

None blocking. One deferred question is recorded under "Assumptions" below
regarding whether wiring the plugin to `externalRefs` is worth a dedicated
future increment.

## Assumptions

- The addendum's example (`externalRefs: [{ provider: azure-devops, type:
  user-story, id, url }]`) describes the target shape once a real
  provider integration exists; it does not mandate that today's
  declarative-only, non-executing plugin already writes it. Wiring is
  only meaningful once `axiom external-sync azure-devops create` (or
  equivalent) actually executes a real Azure DevOps API call and needs
  somewhere to persist the returned work-item id/url — that command does
  not exist yet (confirmed below), so there is nothing to wire today.

## Implementation notes

No implementation subagent was used — findings below justify a
verification-only closure, matching INC-05's and INC-12's precedent that
"roadmap assumes a gap, audit finds none" is a valid, recurring, honest
outcome in this codebase.

### What the plugin actually does today

- `app-plugins.ts` (Spec 0030/D1): a **generic plugin loader**. Reads
  `<projectRoot>/.sdd/<projectName>/app-plugins/*.json`, validates shape
  (`AppPlugin { id, version, displayName, tabs[] }`, each tab has
  `actions[]`, each action has `kind: 'read' | 'local-mutation' |
  'external-mutation'` plus optional `fields[]`), tolerates malformed
  files as warnings (never aborts), rejects duplicate ids. No plugin is
  built-in by this loader itself — default is `[]` if the directory is
  absent.
- `app-plugins-azure-devops.ts` (Spec 0030/D2): a **single hardcoded
  constant**, `AZURE_DEVOPS_PLUGIN: AppPlugin`, declaring 2 tabs
  (`work-items`, `mapping`) and 4 actions (`create-work-item` and
  `register-mapping` as `external-mutation`, `list-work-items` and
  `show-mapping` as `read`). Each action's `command` field names a CLI
  invocation string (e.g. `axiom external-sync azure-devops create`) that
  the frontend would show as a dry-run preview and that `/api/execute`
  would invoke **only after explicit `confirmed: true`**. This file
  contains **zero HTTP calls, zero Azure DevOps SDK usage, zero
  credential handling** — it is pure declarative metadata (a form
  schema), consistent with GATE 0025/0030 ("the app introduces no
  parallel business logic; the bridge declares the contract, it does not
  execute it") stated explicitly in both the file's own header comment
  and `docs/0030-operator-app-plugins-and-external-bridge.md`.
- `app-api.ts`'s `apiGetPlugins` (Spec 0030/D3) is the only consumer:
  `GET /api/projects/:id/plugins` always includes `AZURE_DEVOPS_PLUGIN` in
  the response **unless** a project has its own `azure-devops`-id plugin
  file on disk (filesystem version wins). This endpoint is read-only and
  belongs to the optional `axiom app` local PWA/API server
  (`app.ts`), not to any structural CLI command.
- **The `command` strings the plugin declares do not resolve to a real
  command.** `axiom-external-sync-command` is one of 15 intent commands in
  `@axiom/orchestrator`'s `state-machine.ts` (line ~315), and its
  precondition is literally `notImplemented` — the same stub status as
  `axiom-status-command` and several other intent commands documented in
  `general-spec.md`'s existing "Axiom redesign roadmap" INC-02 entry. No
  `apps/cli/src/commands/external-sync*.ts` file exists. This means: even
  if a user confirmed the `create-work-item` action in the app UI today,
  `executeSubcommand` in `app-api.ts` has no case for an `external-sync`
  workflow id at all (its switch only handles `increment | bug | plan |
  role | qa-e2e`) — the action is UI-reachable but not executable. This is
  a pre-existing, already-documented gap (the orchestrator's own
  `not-implemented` stub posture), not something this increment
  introduces or is scoped to fix.

### Is Axiom Core hard-dependent on this plugin?

No. Import-graph check:

- `apps/cli/src/index.ts` unconditionally registers `registerApp` (from
  `app.ts`), but `app.ts` only starts an **optional local HTTP server**
  (`axiom app`, default port 4567) — a user-invoked, non-mandatory
  command, not something any other command or `axiom doctor` check
  depends on.
- The only import chain touching the plugin files is `app.ts` ->
  `app-api.ts` -> `{app-plugins.ts, app-plugins-azure-devops.ts}`. No file
  under `apps/cli/src/commands/*` other than `app-api.ts` imports either
  plugin file. No package under `Axiom/packages/*` (`doctor`, `workflow`,
  `mcp-tools`, `orchestrator`, `topology`, `skills`, `versioning`, etc.)
  references `app-plugins` or `azure-devops` anywhere (confirmed by
  grepping `Axiom/packages` for both strings — the two `user-workspace`
  hits are unrelated, part of the file named `registry.ts`, not an actual
  reference to the plugin).
- Structural commands (`axiom-increment`, `axiom-bug`, `axiom-plan`,
  `axiom-role`, `axiom doctor`, `axiom mcp`, MCP tool registration) never
  import or call anything from `app-plugins*.ts`.
- Conclusion: **Axiom Core is genuinely plugin-independent.** A project
  that never runs `axiom app` and never drops a file under
  `.sdd/<project>/app-plugins/` never touches this code path at all.
  Addendum §10's core requirement ("Axiom Core debe funcionar sin ellas")
  is already satisfied, not by an explicit conditional-registration
  mechanism, but by the plugin living entirely inside one optional,
  separately-invoked feature (the local PWA/API server) with no reverse
  dependency from core.

### `externalRefs` wiring — disconnected, and correctly so today

- `externalRefs` (INC-06's `ExternalRef` type, INC-11's ADR/decision
  extension) is read/written exclusively through
  `@axiom/workflow/src/artifact-store.ts` and the generic CLI wiring in
  `apps/cli/src/commands/artifact-metadata-cli.ts` (`externalRefs add` /
  `externalRefs list`, provider-agnostic — works for any `provider`
  string, not just `azure-devops`).
- The Azure DevOps plugin (`app-plugins-azure-devops.ts`) does **not**
  reference `externalRefs`, `ExternalRef`, or `artifact-store.ts` in any
  way. Its `field.externalRef?: boolean` is an unrelated, same-named-by-
  coincidence UI flag on a form field (marks which fields belong to "the
  external bridge form"), not a link to the metadata field of the same
  name.
- This is **not a wiring gap worth closing in this pass**, because there
  is nothing real to wire yet: the plugin's `create-work-item`/
  `register-mapping` actions do not execute (see above —
  `axiom-external-sync-command` is `notImplemented`, and `app-api.ts`'s
  `executeSubcommand` has no `external-sync` case). Wiring
  `externalRefs`-writing into an action that cannot run would be
  speculative code with no real caller, which `Axiom.SDD/AGENTS.md`
  explicitly prohibits ("no speculative architecture"). The addendum's
  own worked example (`externalRefs: [{ provider: azure-devops, ... }]`)
  is a target end-state for when a real provider integration exists, not
  a requirement that today's placeholder plugin must already produce it.
- Recorded as a legitimate future follow-up, not a defect: once
  `axiom-external-sync-command` (or a dedicated `external-sync` CLI
  command) is actually implemented against a real Azure DevOps API call,
  that implementation should call `artifact-metadata-cli.ts`'s
  `externalRefs add` path (or the underlying `artifact-store.ts`
  function) to persist the returned work-item id/url, reusing the
  existing generic mechanism rather than inventing a new one. No new
  increment is opened for this now — there is no concrete request to
  implement real Azure DevOps sync, only to verify the current plugin's
  posture.

### Can the plugin be fully absent without breaking anything?

Yes, confirmed two ways:

1. Structurally: `app-plugins-azure-devops.ts`'s only consumer
   (`app-api.ts`'s `apiGetPlugins`) always **falls back to `[]`** plugins
   plus the one hardcoded entry; removing the hardcoded entry would just
   return `[]` when no filesystem plugin exists (`loadAppPlugins`'s own
   documented default), not an error.
2. No other command, doctor check, or MCP tool references this file at
   all (see import-graph check above) — deleting both plugin files would
   only break `apps/cli/src/commands/app-api.ts`'s two import lines and
   `apps/cli/tests/app-plugins.test.ts`, nothing in the rest of the
   monorepo.

## Validation

`No validation command was found.` does not apply here in its literal
"no command exists" sense — `Axiom/package.json` defines real validation
commands (`npm run build`, `npm test`, `npm run doctor`, `npm run
typecheck`), per `Axiom.SDD/AGENTS.md`'s validation discovery order. They
were not run because **no code was changed** in `Axiom` by this
increment — running the suite would only assert pre-existing state, not
validate anything this increment produced, consistent with the same
posture the roadmap increment itself used for its own planning-only
closure.

Best-effort validation performed:

- Read `app-plugins.ts`, `app-plugins-azure-devops.ts`, `app-api.ts`,
  `app.ts`, and `docs/0030-operator-app-plugins-and-external-bridge.md`
  in full.
- Grepped `Axiom/apps/cli/src` and `Axiom/packages` for `app-plugins`,
  `app-api`, `azure-devops`, `AZURE_DEVOPS`, `external-sync`, and
  `externalRef`(s) to build the import graph and confirm no hidden
  dependency exists.
- Read `apps/cli/src/index.ts`'s command-registration block to confirm
  `registerApp` is the only touch point and is itself just an optional,
  user-invoked local server command.
- Read `@axiom/orchestrator`'s `state-machine.ts` to confirm
  `axiom-external-sync-command`'s `notImplemented` status, explaining why
  the plugin's `command` fields do not resolve to working commands today.
- Read `@axiom/workflow`'s `artifact-store.ts` and
  `artifact-metadata-cli.ts` to confirm the actual `externalRefs`
  read/write path and that the plugin does not touch it.

## Result

Every hypothesis in the roadmap's INC-16 entry was confirmed true by
direct reading, with one added, more precise finding: the existing Azure
DevOps "plugin" is a **declarative form schema only** — no HTTP client, no
credentials, no work-item creation logic — and its `command` fields point
to an orchestrator intent (`axiom-external-sync-command`) that is
explicitly `notImplemented`. This is even further from "core dependency"
than the roadmap assumed; it is closer to a UI mockup/contract-declaration
for a future bridge than an active integration. Axiom Core has zero import
or runtime dependency on either plugin file. `externalRefs` remains a
separate, provider-agnostic, already-working mechanism
(`artifact-metadata-cli.ts`), untouched by and correctly unconnected to
this placeholder plugin, because there is no real external-sync execution
path yet to produce a value worth persisting. No code changes were made;
none were warranted.

## General spec integration

Added a short "External plugins (Azure DevOps bridge)" entry to
`Axiom.Spec/general-spec.md` recording: the plugin is declarative-only
(no real Azure DevOps execution), it is structurally isolated from core
(only reachable via the optional `axiom app` server), its `command`
fields reference the still-`notImplemented` `axiom-external-sync-command`
orchestrator intent, and `externalRefs` remains the separate,
already-working, provider-agnostic mechanism it should be wired to only
once (if ever) a real external-sync execution path is built. This is
stable, load-bearing knowledge for any future increment considering a
real Azure DevOps/Jira integration, so it belongs in `general-spec.md`
rather than only in this increment's history.

## Next step recommendation

Proceed to roadmap **INC-17** (optional general command router —
`intent.ts` + `orchestrator`'s intent commands), per the roadmap's own
stated dependency (`INC-06`, `INC-13`, both closed) and sequence. If a
real Azure DevOps integration is ever requested, open it as a new,
separate increment scoped explicitly to implementing
`axiom-external-sync-command`/`axiom external-sync azure-devops *`
end-to-end (API client, credential handling, `externalRefs` wiring at
that point) — do not fold it into INC-17 or treat this closure as
implying that work is queued.
