# Repair legacy Cortex schema drift in Autumn Garage memory

**Date:** 2026-06-04
**Type:** decision
**Trigger:** T1.1, T1.4
**Cites:** doctrine/0006-autumn-garage-is-meta-context, doctrine/0007-branding-system, plans/alchemist, journal/2026-06-04-pr-merged-2246

> While cleaning the generated PR #39 merge note, `cortex doctor` exposed legacy Doctrine and Plan schema drift; the follow-up PR repairs those source files and regenerates state so the coordination repo validates again.

## Context

The generated Cortex PR for Autumn Garage PR #39 started as a post-merge journal cleanup. Running `cortex doctor` against the branch failed with 11 errors unrelated to the generated journal:

- Doctrine 0006 and 0007 used `Status: Active`, which is no longer a valid Doctrine status under SPEC § 3.1.
- `plans/alchemist.md` predated current Plan requirements: canonical `Status`, `Written`, `Author`, `Goal-hash`, `Updated-by`, and the exact required sections.

Leaving those errors in place would make the repo's own validation instruction unactionable. The repair is mechanical schema alignment, not a change to the underlying decisions: both Doctrine entries already represented accepted doctrine, and the Alchemist plan already represented active work.

## What we decided

Repair the legacy schema drift in the same follow-up PR that cleans the generated merge journal:

- Change Doctrine 0006 and 0007 from `Status: Active` to `Status: Accepted`.
- Normalize `plans/alchemist.md` to current Plan frontmatter with `Status: active`, `Written`, `Author`, recomputed `Goal-hash: 979ffb41`, and an `Updated-by` history entry.
- Add the required `## Why (grounding)`, `## Approach`, `## Success Criteria`, and `## Work items` sections by reorganizing existing plan content and making the already-stated success signals explicit.
- Regenerate `.cortex/state.md` from the repaired source files, replacing the stale projection with the current Cortex v1.6.4 output.

## Consequences / action items

- [x] `cortex doctor` exits 0 with no SPEC errors.
- [x] `.cortex/state.md` once again lists the Alchemist plan as the active plan instead of hiding it behind invalid status/frontmatter.
- [x] The remaining `cortex doctor` messages are warnings only: stale map/state migration hints and existing steering-scope warnings in `AGENTS.md` / `CLAUDE.md`.
