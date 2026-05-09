# Unify update / update-all / doctor across the quartet

**Date:** 2026-05-08
**Type:** decision
**Trigger:** T1.1 (diff touches `.cortex/doctrine/`)
**Cites:** doctrine/0008-shared-update-and-doctor-semantics

> Locked in a shared CLI surface (`update`, `update-all`, `doctor`) across Touchstone, Cortex, Sentinel, and Conductor; per-tool implementation issues filed against each repo.

## Context

The four tools shipped their "bring this repo's scaffolding up to date" commands at different moments and ended up with three different verbs:

- **Touchstone:** `update` (single repo) + `sync` (batch) + `doctor` (review-fail-open trends)
- **Cortex:** `sync` (composes `refresh-state` + `refresh-index`) + `doctor` (structural)
- **Sentinel:** none for either update or doctor
- **Conductor:** `refresh-on-commit` (hook) + `refresh-consumers` (batch) + `doctor` (env + providers)

Survey method: `Explore` agent walked each repo's CLI entry point (bash `bin/touchstone`; Python `cli.py` / Click groups for the others) and listed every subcommand with a one-line summary. Same survey enumerated migration commands and doctor scope. Inconsistencies fell into three buckets: command name (`update` vs `sync` vs `refresh-*`), one-vs-many split (per-tool inconsistent), and `doctor` semantics (review trends vs structure vs env vs missing).

Three options weighed for the canonical verb — `update`, `sync`, `refresh`. Picked `update`: plain English, action-oriented, already correct in Touchstone (the most user-facing surface). `sync` reads as bidirectional reconciliation which over-promises. `refresh` is a fine third place but `update` won on familiarity.

For one-vs-many, picked the `<verb>` / `<verb>-all` split over a single `--all` flag. The flag form would have hidden the destructive cross-repo nature inside an option; a distinct subcommand surfaces it.

For `doctor`, picked structural-only semantics. Touchstone's review-trend report belongs in `touchstone review-stats` or `touchstone status`, not `doctor`. Reason: a tool with a green `doctor` should be safe to run; a `doctor` overloaded with metrics blurs that contract.

## What we decided

Doctrine 0008 published. The contract:

- `<tool> update` — current repo, idempotent, no-op when current. Flags: `--dry-run`, `--check`.
- `<tool> update-all` — batch over registered/known projects. Optional per tool.
- `<tool> doctor` — structural install + scaffolding check. Exits non-zero on broken state. No metrics.
- Self-update remains brew-only across all four. No `<tool> upgrade` / `<tool> self-update`.
- Migration commands stay tool-specific and surface from `doctor` when the project needs them.
- Aliases (`touchstone sync`, `cortex sync`, `conductor refresh-consumers`) live ≥2 minor versions with a deprecation note.

Per-tool work tracked as GitHub issues against each tool repo (per the autumn-garage memory note: prefer issues over plans for delegated work). Issue links recorded below once filed.

## Consequences / action items

- [x] Doctrine 0008 written and committed
- [x] Journal entry (this file)
- [ ] Issue filed: autumngarage/touchstone — rename `sync`→`update-all`, move review-trend off `doctor`
- [ ] Issue filed: autumngarage/cortex — add `update` as primary verb; alias `sync`
- [ ] Issue filed: autumngarage/sentinel — add `update` and `doctor`
- [ ] Issue filed: autumngarage/conductor — add `update`; alias `refresh-consumers`→`update-all`
- [ ] Update `.cortex/state.md` once the four issues are filed (add to "Open decisions" or a new "Cross-tool initiatives" line)
- [ ] After all four ship: a single section in autumn-garage README explaining the three verbs once, in lieu of per-tool sections
