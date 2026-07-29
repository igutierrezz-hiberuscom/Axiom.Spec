# INC-20260727-adopt-context-default

Status: closed

## Goal

Migrate a legacy repo's technical context **by default** during adoption — no explicit
`--ingest-context` flag required — so `axiom workspace setup --adopt-spec/--adopt-sdd` also brings
over the project's technical-context docs.

## Context / finding

`workspace setup` adoption only ingested context when `--ingest-context <path>` was passed
(`runWorkspaceAdopt` gated both preview and real ingest on `args.ingestContextPath !== undefined`).
The ingest engine (`ingestContextDocs`) already walks an arbitrary docs tree and classifies each
`.md` into the technical-context taxonomy (architecture/conventions/operations/testing/
integrations/references), but it must be pointed at the context ROOT — NOT the repo root, which
would wrongly ingest spec-artifact folders. Legacy KVP25 keeps its technical context in a
`context/` dir (`context/{architecture,backend,frontend}/*.md`, 164 files).

## Scope

- New exported helper `autoDetectContextSource(adoptSpecSourcePath?, adoptSddSourcePath?)`
  (`apps/cli/src/commands/workspace-adopt.ts`): when `--ingest-context` is absent, look for a
  conventional context dir inside the adopted legacy repo(s) — prefer `technical-context/`, then
  `context/`; spec repo before sdd repo. Returns the subdir (never the repo root) or `undefined`.
- `runWorkspaceAdopt` computes `effectiveIngestContextPath = args.ingestContextPath ?? autoDetected`
  and uses it at both ingest sites (dry-run preview + real run). An explicit `--ingest-context`
  always wins. The dry-run/real report notes when the source was auto-detected.

## Non-goals

- No change to `ingestContextDocs` / the classification (already handles arbitrary trees).
- Never scan the repo ROOT (would ingest spec artifacts). Only conventional subdirs.
- If no context dir exists, adoption proceeds without context ingest (prior behavior).

## Acceptance criteria

- Adopting a legacy repo with a `context/` (or `technical-context/`) dir migrates its docs with no
  `--ingest-context` flag; the report shows the auto-detected source.
- An explicit `--ingest-context` overrides the auto-detection.
- No context dir → no ingest, no error.

## Result (closed 2026-07-27)

Implemented as above. **Validation:** `tsc -b` clean; 5 new `autoDetectContextSource` unit tests in
`workspace-adopt.test.ts` (prefers technical-context, spec-before-sdd fallback, dir-not-file,
undefined when none) + existing adopt suite green. **Real E2E (2026-07-27):** `workspace setup
--adopt-spec <KVP25/Kvp.Spec>` WITHOUT `--ingest-context` → report "Context: auto-detectado
'…\Kvp.Spec\context'"; real run → `contextCreated: 164`, `technical-context/` populated across
architecture/operations/references/testing/indexes. Integrated into
`06_Integraciones_y_Capacidades.md`.
