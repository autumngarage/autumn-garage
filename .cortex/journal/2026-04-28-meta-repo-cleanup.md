# Meta-repo cleanup — autumn-garage shrunk to meta-context only

**Date:** 2026-04-28
**Type:** decision
**Trigger:** T1.1 (touches `.cortex/plans/` — many deletions) + T1.4 (file deletion >100 lines per file × 15 files)
**Cites:** doctrine/0006-autumn-garage-is-meta-context, journal/2026-04-28-sentinel-autonomous-engineer-plan

> Cleaned up 14 accumulated plans from autumn-garage's `.cortex/plans/`. 8 moved to their owning quartet/autumn-mail repos via PRs (sentinel#91, conductor#95, touchstone#85, autumn-mail#9). 7 outrider-intel/vanguard plans deleted outright — autumn-garage shouldn't write planning artifacts into a customer's repo, and the customer has its own conventions (`docs/plans/`, no Cortex). End state: autumn-garage's `.cortex/plans/` empty, principle codified in Doctrine 0006.

## Context

The 2026-04-28 conversation about Hermes Agent surfaced a discipline drift: autumn-garage had accumulated 16 plans in `.cortex/plans/`, most of them tool-specific or about a customer's product (outrider intel's vanguard/outrider repos). The user articulated the principle: "Autumn-garage reads upwards and writes downwards. We don't need to store anything in Autumn Garage proper, aside from an understanding of each tool."

## What we decided

1. Doctrine 0006 codifies: autumn-garage holds doctrine + state + journal + templates + shared infra only. All plans — even cross-cutting — live in the primary owner's repo. Cross-cutting decisions become doctrine entries here.

2. Plan dispositions:

| Plan | Disposition |
|---|---|
| `sentinel-conductor-migration.md` | → sentinel#91 (Slice B revised for subprocess seam) |
| `sentinel-autonomous-engineer.md` | → sentinel#91 (cross-cutting, primary owner is sentinel) |
| `sentinel-codex-identifier-rename.md` | → sentinel#91 (superseded; historical) |
| `sentinel-cortex-t16-integration.md` | → sentinel#91 (active) |
| `conductor-bootstrap.md` | → conductor#95 |
| `conductor-http-tool-use.md` | → conductor#95 (shipped) |
| `llm-provider-additions.md` | → conductor#95 (superseded) |
| `local-llm-provider-alignment.md` | → conductor#95 (superseded) |
| `touchstone-conductor-integration.md` | → touchstone#85 (active) |
| `autumn-mail-dogfood.md` | → autumn-mail#9 (active) |
| `complete-the-migration.md` | DELETED — outrider intel domain |
| `cutover-runbook.md` | DELETED — outrider intel domain |
| `full-vanguard-outrider-separation.md` | DELETED — outrider intel domain |
| `separation-finish-line.md` | DELETED — outrider intel domain |
| `vanguard-db-ownership.md` | DELETED — outrider intel domain |
| `vanguard-execution-flywheel-stage-2-db-ownership.md` | DELETED — outrider intel domain |
| `vanguard-execution-vision.md` | DELETED — outrider intel domain |

3. Outrider Intel relationship clarified: vanguard and outrider are outrider intel's product (a *customer* of autumn-garage). Autumn-garage shouldn't write planning artifacts into a customer's repo. Vanguard/outrider use `docs/plans/`, not Cortex. The customer's product work stays in their repos; their feedback against the quartet tools comes back here as journal entries (dogfood signal).

## Consequences / action items

- [x] Doctrine 0006 written.
- [x] 4 quartet/autumn-mail PRs opened: sentinel#91 (4 plans), conductor#95 (4 plans), touchstone#85 (1 plan), autumn-mail#9 (1 plan).
- [x] Outrider intel customer relationship saved to memory.
- [ ] After target PRs merge, this cleanup PR's deletions take effect on autumn-garage main.
- [ ] Future: when state.md next refreshes, prune any references to the moved/deleted plans; rely on cross-repo cites + tool `.cortex/state.md` instead.
