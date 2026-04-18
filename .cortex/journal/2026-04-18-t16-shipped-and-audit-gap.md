# T1.6 shipped — plus a real Cortex audit dependency surfaced

**Date:** 2026-04-18
**Type:** decision
**Trigger:** T2.1
**Cites:** plans/sentinel-cortex-t16-integration, https://github.com/autumngarage/sentinel/pull/74, journal/2026-04-18-r2-wizards-shipped

> Sentinel #74 ships T1.6 — cycle-end writes into `.cortex/journal/` when `.cortex/` is detected. 34 new tests, all 12 plan success criteria mapped (8 automated, 1 real-cortex-gated, 3 deferred or manual). Real coordination dependency surfaced in the process: Cortex's `doctor --audit` doesn't classify T1.6 fires yet.

## The dependency we didn't have visibility on

Plan criterion #3 says: *"cortex doctor --audit on the same repo matches the T1.6 fire to the produced journal entry."*

The agent discovered during implementation that this can't happen today. Cortex Phase B shipped T1.1/T1.5/T1.8/T1.9 classification in `cortex doctor --audit`. T1.2/T1.3/T1.4/T1.6/T1.7 are deferred per cortex's own `.cortex/state.md` — they need runtime session state or per-commit diff parsing that's out of Phase B's scope.

So T1.6 writes journal entries that validate under `cortex doctor` (frontmatter + shape are correct) but are not yet *matched to a trigger fire* in `cortex doctor --audit`. The entries are valid memory; they're just not yet part of the enforcement loop.

## Why this is a healthy finding, not a defect

If Sentinel had written T1.6 entries that Cortex silently accepted without matching them to fires, we'd have a false sense of end-to-end validation. The gap being explicit — and documented in the agent's PR report, not hidden — means:

1. The next Cortex work (T1.6 classification in `--audit`) has a clear motivating consumer.
2. The autumn-mail dogfood cycle will produce a T1.6 journal entry that's *valid Cortex memory* but waits for the audit layer to become enforced. That's a correct state, not a bug.
3. The file-contract composition rule (Doctrine 0001 / Cortex 0002) is working — the two tools compose through the journal format independently. Sentinel doesn't need Cortex's audit to ship; Cortex's audit will pick up Sentinel's entries when it's ready.

## What's now on Cortex's plate

Added to TODOs.md: "`cortex doctor --audit` classifies T1.2/T1.3/T1.4/T1.6/T1.7 fires." This is a Cortex Phase C extension. Sentinel's T1.6 writes give Cortex a real corpus to validate against.

## State of the trio after T1.6

**12 PRs open across Touchstone (4), Cortex (4), Sentinel (4).** Four-round build-out on the tool side is complete. Remaining:

1. **User reviews + merges the 12 PRs.** Real work.
2. **Regression pass** against `brew upgrade`d binaries once merges land.
3. **Autumn-mail first real cycle** — flip the gws-integration proposal, `sentinel work --budget $3`, verify that the full loop (scan → plan → execute → review → T1.6 write → `cortex doctor` clean) works end-to-end against an unmerged feature-branch install.
4. **Cortex `doctor --audit` T1.6 classification** — the dependency surfaced above. Can ship independently; unblocks the last automated gate on the T1.6 plan.

## What worked — parallel agent dispatch, take four

Four rounds of 3 parallel agents each (plus T1.6 as a single larger dispatch) produced 12 clean PRs in a few hours of agent wall-time. Cost to conversation-level context: minimal — each agent got a self-contained brief and returned a brief report. Pattern is legible enough to repeat.

One observation worth codifying: **specifying the plan *as an artifact* first** (the T1.6 plan went into `plans/` before dispatch) changed the agent's behavior. The agent's PR report cited the plan's criterion numbers, flagged deviations explicitly against the plan, and treated the plan as the contract rather than the brief. That's much stronger than just passing scope inline.

Generalizable rule: for any non-trivial cross-tool work, **write the plan first, dispatch second.** The plan is the spec; the brief is the dispatch command.
