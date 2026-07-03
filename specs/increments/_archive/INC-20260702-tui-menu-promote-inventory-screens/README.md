# Increment: Promote mcp-inventory/memory-inventory to first-class TUI menu items

Status: pending (not started — deferred placeholder)
Date: 2026-07-02

## Goal

Promote `mcp-inventory` and `memory-inventory` from CLI-flag-only entry
points (`--mcp-inventory`, `--memory-inventory` or equivalent) to
first-class items in `@axiom/tui`'s `MENU_ITEMS` (`packages/tui/src/
router.ts`), consistent with how `topology`/`projects`/`model-list` are
already reachable both by flag and by menu.

## Context

Raised as **OQ1** during INC-04's migration-engineer audit
(`Axiom.Spec/specs/increments/INC-20260702-tui-shell-detection-reconcile/README.md`).
The user explicitly deferred this out of INC-04's scope (INC-04 is
detection heuristics + menu relabeling only, not new screen wiring) but
asked that it be tracked as its own increment rather than dropped.

## Scope

- Add `mcp-inventory` and `memory-inventory` to `MENU_ITEMS`.
- Confirm both screens render correctly when reached via the menu (today
  they are reachable only by CLI flag; the audit noted they may currently
  be under-wired/placeholder-only — verify before assuming they're
  menu-ready).

## Non-goals

- No redesign of the two screens' content.
- No other menu items or detection-heuristic work (that's INC-04's scope).

## Status

Not started. This is a placeholder to preserve the finding; scope it
properly (migration-engineer audit of the two screens' actual
implementation state) when picked up.
