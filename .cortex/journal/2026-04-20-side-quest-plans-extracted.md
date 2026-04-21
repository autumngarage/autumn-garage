# Two pre-existing cross-tool drifts extracted into their own coordination plans

**Date:** 2026-04-20
**Type:** decision
**Trigger:** T1.1
**Cites:** plans/sentinel-codex-identifier-rename, plans/local-llm-provider-alignment, doctrine/0003-llm-providers-compose-by-contract, journal/2026-04-20-llm-provider-contract, plans/llm-provider-additions

> Writing `integration/providers.md` surfaced two pre-existing cross-tool drifts that violate Doctrine 0003 §1 (matching identifiers across tools): Sentinel labels Codex as `openai` while Touchstone calls it `codex`; the `local` identifier means "Ollama HTTP" in Sentinel and "generic subprocess command" in Touchstone. Promoted both from "deferred follow-up notes inside `plans/llm-provider-additions.md`" to first-class coordination plans. Doctrine ships clean — known violations have owners and scope, not silent debt.

## Context

Per the user's encouragement to fix systemic issues as side quests, after writing the providers reference table I had two options for the surfaced drifts:

1. Leave them as bullet points in `plans/llm-provider-additions.md`'s "Follow-ups (deferred)" section. Cheap, but they accumulate as deferred items inside an unrelated plan.
2. Promote them to dedicated coordination plans with their own scope, success criteria, and work items. More work upfront but matches the autumn-garage protocol — every cross-tool decision becomes a real plan.

Doctrine 0003 itself argues for option 2: the contract is real, the violations are real, and the fix is real. Letting them live as inline bullets would be exactly the "implicit coordination" anti-pattern Doctrine 0001 names ("decisions drift and the integration contract ossifies only after real divergence has occurred").

The two drifts also have very different shapes:

- **Sentinel codex/openai rename** is mechanical. The fix is a known pattern (rename + backward-compat alias + deprecation window). One Sentinel PR. Ready to start whenever the user picks the queue position.
- **Local provider alignment** has real design uncertainty. Sentinel's `local` is opinionated (Ollama HTTP); Touchstone's `local` is generic (subprocess escape hatch). They're two different abstractions sharing a name. The plan opens with three options (A: split into two identifiers; B: converge on HTTP; C: converge on subprocess) and asks the user to pick before any per-tool work begins. The plan is currently structured for Option A as the recommendation but is rewriteable.

## What we decided

1. **Promote both drifts to first-class coordination plans.** `plans/sentinel-codex-identifier-rename.md` and `plans/local-llm-provider-alignment.md` written today.
2. **Update `plans/llm-provider-additions.md`** to point at the new sibling plans rather than carry the work as inline follow-ups.
3. **Do not start implementation yet.** The user should pick the order: Kimi rollout (the original ask), codex rename, and local alignment are three workstreams; implementation queue is the user's call.
4. **The local-alignment plan is decision-pending.** Phase 1 of that plan is "user picks Option A, B, or C." Implementation work items are written provisionally for Option A but are explicitly rewriteable.

Considered and rejected:

- **Inline follow-ups only.** Doctrine 0003 just shipped; the protocol it lives under (autumn-garage's CLAUDE.md + Cortex Protocol §4) treats cross-tool decisions as plan-worthy. Leaving them as bullets would be the exact pattern the doctrine is meant to prevent.
- **Bundle the codex rename into the Kimi rollout plan.** Tempting because both touch Sentinel's provider layer. Rejected because it conflates two unrelated changes and would make the Kimi PR much wider in scope. One workstream per plan.
- **Defer the local-alignment decision.** Could ship Kimi + codex rename and leave local alignment as a future item. Rejected because the doctrine ships with a known violation; recording it as a real plan with an open decision is more honest than burying it.

## Consequences / action items

- [x] Created `plans/sentinel-codex-identifier-rename.md`.
- [x] Created `plans/local-llm-provider-alignment.md`.
- [x] Updated `plans/llm-provider-additions.md` Follow-ups section to reference the new plans.
- [ ] User decides the implementation queue: Kimi rollout first, codex rename first, or local-alignment decision first?
- [ ] User decides the local-alignment shape (Option A / B / C). Recommended: Option A. Until decided, no per-tool implementation on the local front.
- [ ] Once implementation order is set, update `state.md` with the active workstream(s).
