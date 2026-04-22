# Touchstone × Conductor integration plan ratified — build begins

**Date:** 2026-04-21
**Type:** decision
**Trigger:** T1.1 (diff touches .cortex/plans/) + T1.3 (plan status proposed → active) + T2.1 (user phrased a decision)
**Cites:** plans/touchstone-conductor-integration, plans/conductor-bootstrap, doctrine/0004-conductor-as-fourth-peer, doctrine/0003-llm-providers-compose-by-contract

> Human ratified the ideal-state Touchstone × Conductor integration plan in full, directing execution across the ~8-week roadmap with no short-term hedging. Plan flipped to Status: active; implementation begins in Conductor.

## Context

Over the course of one session (2026-04-21), the integration plan went through five design iterations in dialogue:

1. **Initial additive v0.1** — `auto` as a new cascade entry alongside existing codex/claude/gemini/local adapters.
2. **Ideal-state rewrite** (human: "I don't need a short-term version") — Conductor becomes the *only* LLM abstraction Touchstone knows. All four per-provider bash adapters deleted. Touchstone 2.0.0.
3. **Quality-tier axis** (human: "always use the best model") — router learns `prefer = best | cheapest | fastest | balanced`; providers declare `quality_tier`; cost scoring includes thinking tokens.
4. **Concierge init C8** (human: "go 1 by 1, make sure that is an experience") — `conductor init` walks each of 6 providers with copy-pasteable install commands, inline smoke tests, skip/resume.
5. **Effort axis** (human: "claude to use max effort", then: "any model") — orthogonal global dial, symbolic levels mapped per-provider.
6. **Reliability promises** (human: "user configurable + make auto mode actually work") — added C9 (validation, effective-config inspector, dry-run, env-var parity) and C10 (health tracking, explainable scoring, graceful degradation, override-feedback loop).

Plan reached 10 required Conductor surface additions (C1–C10), 5 sequenced stages, ~8 weeks of engineering estimated with Stage 3 (HTTP tool-use loop) as the single largest chunk (~4 weeks).

Human then committed: **"let's build it all. let it rip brother"** — with `/effort max` engaged in the harness.

## What we decided

- **Ratify the plan as written.** No further scope iteration. Status: proposed → active.
- **Begin implementation in Conductor immediately.** Stage 1 (shell-out `exec`, capability declarations, quality tiers, effort axis, route logging, basic C9/C10) is the unlock that Touchstone 2.0 migration in Stage 2 depends on.
- **Stage 3 (HTTP tool-use loop) is explicitly out of this session's scope.** ~4 weeks of focused work; deserves its own dedicated plan before implementation.
- **Work on a feature branch in conductor, not main.** Precedent set by PRs #1–#4. Feature branch → PR → squash-merge.
- **Cross-tool coordination decisions continue to journal here (autumn-garage).** Single-tool Conductor decisions (e.g., exact internal module layout) journal in `~/Repos/conductor/.cortex/`.

## Consequences / action items

- [x] Flip `plans/touchstone-conductor-integration.md` Status to active.
- [x] This journal entry (T1.3 + T1.1).
- [ ] Feature branch in conductor: `feat/v0.2-exec-and-preferences`.
- [ ] Stage 1 implementation in batches:
  - [ ] Provider capability declarations (quality_tier, supported_tools/sandboxes, cost fields, effort_map, supports_effort) on all 5 providers
  - [ ] Router upgrade: `prefer` modes, effort application, session-local health tracking, explainable RouteDecision
  - [ ] `conductor call` new flags (--prefer, --effort, --tools, --sandbox, --exclude, --log-route)
  - [ ] New `conductor exec` subcommand for shell-out providers (claude/codex/gemini)
  - [ ] Route log output (default terse, --verbose for full ranking)
  - [ ] Graceful 5xx fallback (one-hop)
  - [ ] Config validation with fix-it hints
  - [ ] `conductor config show` effective-config inspector
  - [ ] `conductor route --dry-run`
  - [ ] Concierge init rewrite (C8)
- [ ] Ship Conductor v0.2 release.
- [ ] Stage 2 (Touchstone migration PR) after v0.2 stabilizes — separate session.
- [ ] Dedicated plan for Stage 3 (HTTP tool-use loop) before Stage 3 implementation.

## What's not in scope for this execution arc

- Sentinel migration to Conductor (separate plan, lands post-Stage-3 because Sentinel's coder role needs HTTP tool-use working).
- Conductor brew tap publication.
- Actual quality benchmarking of providers (tiers remain declared, not measured).
- Persistent cross-session health state.
- ML-based or learned routing.
