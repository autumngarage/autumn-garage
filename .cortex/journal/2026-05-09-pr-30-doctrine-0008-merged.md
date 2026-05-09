# PR #30 merged - Doctrine 0008 shared update semantics

**Date:** 2026-05-09
**Type:** pr-merged
**Trigger:** T1.9
**Cites:** doctrine/0008-shared-update-and-doctor-semantics, journal/2026-05-08-shared-update-semantics
**Merge-commit:** 11b57f25a0fe650c5102343271863918c2f20b78
**Branch:** docs/doctrine-0008-shared-update-semantics

> PR #30 shipped the shared `update`, `update-all`, and `doctor` command contract for the quartet and ratified the per-tool implementation issue links.

## What shipped

- Doctrine 0008 now defines the cross-tool CLI surface for Touchstone, Cortex, Sentinel, and Conductor.
- The accompanying decision journal records the survey method, command-name rationale, and follow-up issue links.
- Follow-up implementation issues are filed and linked for all four tools.
- The PR review thread about unfiled follow-up issues was addressed and resolved before merge.

## Closes / advances

- **Plans:** none.
- **Doctrine:** new: doctrine/0008-shared-update-and-doctor-semantics.
- **Journal linkage:** ratifies journal/2026-05-08-shared-update-semantics.

## Follow-ups (deferred to future work)

- [ ] Ship per-tool implementations: touchstone#257, cortex#238, sentinel#118, conductor#303.
- [ ] Add a single autumn-garage README section for the three shared verbs after all four implementations land; tracked from journal/2026-05-08-shared-update-semantics.
