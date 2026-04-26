---
date: 2026-04-26
type: decision
trigger: T1.1 (.cortex/plans/ touched)
load-priority: normal
---

# Separation finish-line plan + B.2 detail plan

## Context

Earlier today the all-nighter wave shipped 44 PRs covering everything that didn't require Railway state changes: Shape 3 proposal extension, candidate_instruments populated across kalshi+polymarket+options, cluster_registry isolation fix, in-process flywheel deletion, vanguard.contract mirrors at v0.4.0/schema 2.1.0, AST cluster-boundary tests both directions, comprehensive CONTRACT.md rewrite. By the end the master plan still had 16 pending tasks across the parking lot.

User asked for an honest ROI review against the separation goal. The answer: only 3 deferred items move separation forward — B.2 (vanguard owns its DB), P2.7 (drop the dep), and the Railway env cleanup. Everything else is post-separation work.

User then asked for a clean prioritization and next-steps docs in each repo.

## What I did

1. **Wrote `.cortex/plans/separation-finish-line.md`** — consolidates the remaining critical-path work into 7 stages (SF-1 through SF-7). Tonight's safe code-only work (SF-1/2/3) shrinks the AST cluster-boundary allow-list from ~167 entries to B.2-only. Daylight work (SF-4/5) needs Railway DB provisioning and is the genuine final dep-drop. Documentation (SF-6/7) ships tonight.

2. **Wrote `.cortex/plans/vanguard-db-ownership.md`** — the detailed B.2 plan, 3 sub-stages (provision + dual-write, cutover, drop). Captures the operator runbook the user picks up in daylight: provision Railway Postgres, set `DATABASE_URL_VANGUARD`, vendor the write paths, dual-write soak, cutover, drop the dep. Estimated 3-4 working days of execution + 2-3 days of soak.

3. **Updated task graph** — created tasks #69-76 for SF-1 through SF-8, set blocks/blockedBy so the deferred parking-lot tasks (#44 legal Tier 1, #48 streaming, #49 DX, #50 SLA, #60 Layer 2 invariants, #62 cortex install, #63/64 customer auth/onboarding) explicitly block on SF-5. Existing #40 and #61 redirected to point at the new plans.

4. **Spawned agents in parallel:**
   - SF-1 (vanguard, C.1 finalize): vendor `outrider._platform.trade_math`, delete `trading_profile` usages.
   - SF-2 (vanguard, C.2 finalize): swap `outrider.api.*` → `vanguard.contract.*` mirrors.
   - SF-3 (vanguard, A.1 finalize): delete autolab/quant/council/forge/agents/flywheel/strategies imports.
   - SF-6 (vanguard, docs): NEXT_STEPS.md PR.
   - SF-7 (outrider, docs): NEXT_STEPS.md PR.
   - URGENT sweep (vanguard): `outrider._platform` → `outrider.platform` rename mechanical fix because outrider PR #39 (just merged) renamed the directory and vanguard pins outrider@main.

## Why split the plan from the master plan

The parent `full-vanguard-outrider-separation.md` describes the architectural target and the workstream taxonomy. After tonight's wave, the parent has too many "done" entries to be useful as a roadmap. The finish-line plan is a focused punch list with concrete tasks, sequencing, and a single definition-of-done. When SF-5 ships and the dep drops, the finish-line plan flips to Status: shipped and the parent plan can be retired or kept as historical context.

## Honest read on what's left

After tonight's SF-1/2/3 + sweep land:
- AST allow-list contains ~25 entries, all B.2 (DB).
- Vanguard cannot drop the dep because vanguard's runtime writes operational state into outrider's Postgres.
- Everything else is documented and parked.

After the user provisions Railway DB and B.2 ships in daylight:
- Allow-list goes empty.
- Dep drops in SF-5 (mechanical PR).
- Both repos work independently.

Total wall-time to true separation: ~4-5 days of execution from when Railway DB is provisioned, including soak windows.

## What I deliberately did NOT do

- Did not pursue WS4 streaming, WS5 DX, WS6 SLA, WS11 Layer 2 invariants, WS12/13 customer auth — these are post-separation excellence work. Doing them now would muddy the boundary while the dep is still in place.
- Did not touch legal Tier 1 posture — real concern but orthogonal to code separation.
- Did not pre-emptively split the AST allow-list shrink across more agents — three is the right granularity (one per category) and matches the parallel-agent vanguard contention model from earlier tonight.
