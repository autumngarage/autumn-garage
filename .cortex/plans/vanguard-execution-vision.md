---
Status: Active (post-migration roadmap)
Owner: vanguard
Parent: `.cortex/plans/complete-the-migration.md` (gates this — vision activates once dep drops)
Created: 2026-04-26
Goal-hash: (recompute with cortex doctor)
---

# Vanguard execution vision

> **Vanguard becomes the institutional-grade execution engine for prediction-market and options trading. Not Jane Street's microsecond stack — that's the wrong reference for second-to-minute event-driven trading. The right reference is hedge-fund execution rigor (Renaissance, DE Shaw, AQR) applied to vanguard's actual problem space: take research alpha as input, translate to positions with maximum information preservation, minimum slippage, full auditability.**

## The shape of "great"

Twelve pillars. Each one is independently shippable. Sequenced together over ~4 quarters post-migration.

The unifying principle: **every trade vanguard makes should be debuggable, replayable, and provably consistent with the gates that let it through.** That's the institutional bar. Mature quant firms hit it; retail systems don't.

---

## Pillar 1 — Single-currency contract + zero shared state with research

**Status: shipped (post-migration).**

The architectural foundation. Proposal in, position out, outcomes back. No DB sharing, no in-process Python coupling, no shared libraries above stdlib. Vanguard pip-installs nothing from outrider; outrider knows nothing about vanguard's process.

This is the load-bearing invariant the migration was for. Every other pillar is built on top.

---

## Pillar 2 — Stochastic optimal sizing (beyond Kelly)

**Today:** fractional Kelly + edge-tier multipliers + position-cap gates.

**Advanced version:** multi-step stochastic optimal control. Almgren-Chriss-style framework adapted for binary markets. Sizing at entry conditions on:
- Expected hold time
- Expected slippage path (depth × time)
- IM/MM evolution under adverse moves
- Correlated-position concentration risk

**Mechanism:** solve the HJB equation offline per (edge, vol, liquidity, time-horizon, correlation-tier) bucket. Runtime is a lookup table. Computational cost amortized.

**Why it matters:** Kelly assumes infinite liquidity and instant fills. Real prediction markets have neither. Sizing that conditions on microstructure regularly beats Kelly by 10-20% Sharpe in backtests.

---

## Pillar 3 — Bayesian belief evolution during hold

**Today:** mechanical exits (stop-loss, take-profit, time-exit) + LLM exit-reasoning at fixed checkpoints.

**Advanced version:** a Bayesian network per held position. State updates as evidence arrives:
- Market price moves (likelihood-weighted update)
- Time decay (prior shrinks toward ambiguity)
- News shocks (large-deviation update)
- Agent updates from outrider (via outcome events)
- Correlated-market moves

**Mechanism:** posterior probability is the input to exit logic. `POST /v1/exit-reasoning` is one source of evidence; vanguard fuses it with mechanical observations using a small Bayesian filter (probably particle-based for non-Gaussian likelihoods).

**Why it matters:** mechanical exits are conservative — they exit on price triggers regardless of fresh information. Bayesian exits exploit the information vanguard receives between entry and resolution.

---

## Pillar 4 — Cross-venue arbitrage + portfolio Greeks

**Today:** route per-`candidate_instruments` per proposal; positions managed independently.

**Advanced version:**
- **Cross-venue arb detection.** When kalshi and polymarket both list an event, prices diverge briefly. Detect arb in real time; execute paired trades that lock in spread.
- **Portfolio-level Greeks.** Continuous Δ, Γ, Θ, ν computation across all options positions. Risk gates look at *aggregate* Greek concentration, not just per-position. Auto-hedge when portfolio Δ exceeds threshold.

**Why it matters:** cross-venue arb is execution-only alpha — outrider produces zero of it. Greeks management is what separates "trading options" from "structured-product manufacturing."

---

## Pillar 5 — Smart order routing with latency + fill-rate awareness

**Today:** route per `candidate_instruments`; order type is per-strategy default.

**Advanced version:** per-venue execution policy that learns from realized fills.
- Track per-(venue, market, time-of-day) fill rate, partial-fill probability, latency-to-fill.
- Order type chosen per order based on those tracked metrics: aggressive limit on deep books, post-only-with-timeout on thin, IOC for urgency.
- Auto-tune post-only timeout window from rolling fill-rate stats.

**Why it matters:** prediction markets have heterogeneous liquidity profiles. Treating them uniformly leaves alpha on the table. This is firm-knowledge — vanguard learns it; outrider can't.

---

## Pillar 6 — Pre-trade simulator + deterministic replay

**Today:** orders go from sizing → broker. No simulation step.

**Advanced version:**
- **Pre-trade simulator.** Every order, before broker submission: simulate against current snapshot (book depth, recent trades). Predict fill price + slippage. If predicted-vs-target diverges beyond threshold, halt + alert.
- **Deterministic replay.** Any production day reconstructible from outbox events + market snapshots + cached LLM responses. Production incidents become unit tests.

**Why it matters:** these two together transform engineering culture. Bugs become reproducible. New strategies validated against historical scenarios before live deploy. Postmortems run from canonical replays. This is what makes Jane Street's culture possible — every prod incident is a replayable scenario.

---

## Pillar 7 — Property-based testing of risk math

**Today:** example-based tests of risk gates.

**Advanced version:** Hypothesis-style property tests. For any portfolio state generated by the property tester, no risk gate returns "approve" when invariants are violated.

Plus mutation testing: when you mutate the risk-math code, do tests fail? If not, the test isn't enforcing the invariant — gap visible immediately.

**Why it matters:** risk-math bugs cost capital. Property + mutation testing is what professional quant firms use to keep risk math correct as it evolves. Cheap to set up, expensive not to have.

---

## Pillar 8 — Slippage + edge attribution loop (execution flywheel)

**Today:** outcomes are POSTed to outrider; vanguard doesn't independently learn from them.

**Advanced version:** every fill produces a delta (`realized edge - forecast edge`). Bucket by (strategy, instrument, time-of-day, agent-of-origin, edge tier, market quality, venue). Statistical learning loop:
- Sizing adjusts: if instrument X consistently realizes 80% of forecast edge, future sizing on X shrinks proportionally.
- Routing adjusts: if venue Y has slipping fill rates, weight away.
- Strategy promotion: paper-to-live thresholds adjust based on realized vs paper performance.

**Why it matters:** this is vanguard's parallel to outrider's research flywheel — fundamentally different signal (microstructure realization) than what outrider learns (agent calibration). Each system flywheels its own concerns.

---

## Pillar 9 — Real-time observability + circuit breakers

**Today:** application logs.

**Advanced version:**
- **Per-strategy dashboard:** realized P&L, hit rate, Sharpe, drawdown, fill latency, slippage. Updated in real time.
- **Adaptive circuit breakers:** Sharpe drops 30% over rolling 100 trades → auto-pause that strategy. Aggregate fill latency on a venue spikes 3σ → route around it. Anomalous order rejections → flag operator.
- **Strategy-level kill switches** with documented rollback procedures.

**Why it matters:** institutional execution requires real-time visibility into "is this still working." Dashboards aren't decoration — they're the operator's primary feedback loop.

---

## Pillar 10 — Immutable audit trail

**Today:** ad-hoc logging.

**Advanced version:** every decision logged with full upstream context — which proposal, which gates fired, why-passed-or-rejected, the sizing math, the broker call. SHA-256-chained so the log can't be tampered. Includes: decisions, configurations, deploys.

**Why it matters:** passes regulatory examinations cleanly. Makes incident postmortems possible. Required for any future scaling beyond personal capital.

---

## Pillar 11 — Treasury + capital management

**Today:** capital allocated per-broker semi-statically.

**Advanced version:** unified capital view across kalshi + schwab (+ future polymarket). Track:
- Margin utilization per venue
- Capital efficiency per dollar deployed
- Unrealized P&L by venue and strategy
- Hard limits on aggregate exposure across venues, even when individual venue gates are fine

Auto-rebalance when one venue is starved or over-concentrated.

**Why it matters:** running capital across venues without unified treasury management leaves money idle in one place while another venue is starved.

---

## Pillar 12 — Strategy promotion via paper-to-live

**Today:** ad-hoc.

**Advanced version:** codified criteria.
- N consecutive sessions with paper Sharpe ≥ X
- Per-strategy paper sample size ≥ Y trades
- Manual sign-off step (audit log of who promoted what when)
- Automatic demotion: live performance falls below paper baseline → demote back to paper for re-validation

**Why it matters:** stops "interesting backtest" from becoming "live capital at risk" until evidence is strong. Protects against regime-change drift.

---

## Sequencing

### Quarter 1 (immediately post-migration) — foundations

Goal: vanguard becomes a system that can debug itself.

- **Pillar 10** — immutable audit trail (foundational; everything else benefits)
- **Pillar 6** — deterministic replay (paired with audit trail; outbox events + snapshots → replayable)
- **Pillar 7** — property-based risk math tests
- **Pillar 9** — real-time observability dashboard

### Quarter 2 — execution intelligence

Goal: vanguard learns from what just happened.

- **Pillar 8** — slippage + edge attribution loop (depends on audit trail from Q1)
- **Pillar 5** — smart order routing (depends on attribution data)
- **Pillar 12** — codified paper-to-live promotion (depends on attribution + observability)

### Quarter 3 — advanced sizing + cross-venue

Goal: vanguard does what retail can't.

- **Pillar 2** — stochastic optimal sizing (HJB-derived lookup tables)
- **Pillar 4** — cross-venue arb + portfolio Greeks (gates on outrider's polymarket coverage WS1 shipping)
- **Pillar 11** — unified treasury management

### Quarter 4 — Bayesian + adversarial

Goal: vanguard exploits information that arrives between entry and exit.

- **Pillar 3** — Bayesian belief evolution during hold
- Adversarial pattern detection (kalshi microstructure manipulation around big news) — sub-pillar of Pillar 9

### Year 2

Continuous improvement. Each pillar gets refinement passes. New pillars emerge from production observations. The Year 1 plan delivers the institutional bar; Year 2 is what separates from peers.

---

## Operating principles

These apply across all pillars and frame every PR:

1. **Type safety wins.** Every contract, every state machine, every risk gate boundary: pydantic-strict (`extra="forbid"`) on the wire, dataclass-based with type hints internally. Bugs caught at boundary > bugs caught in tests > bugs caught in prod.

2. **Idempotency is not optional.** Every external call (broker, outrider HTTP, DB write) carries a request-id. Retries don't double-send. Outbox events are deduped on outcome_id.

3. **Reconciliation runs daily.** Vanguard's view of positions matches the broker's view; mismatches alert immediately. Audit log matches reality matches paper trading.

4. **No magic flags in prod.** Feature flags are visible, audited, rollback-able. No silent A/B tests on real money.

5. **Fail loud, fail fast, fail safe.** Risk gate ambiguity defaults to "reject the trade." Broker timeout defaults to "halt the strategy." Reconciliation mismatch defaults to "page the operator."

---

## What this plan does NOT cover

- **Outrider improvements.** Per-customer auth, streaming API, observability/SLA — those are outrider's roadmap (`outrider/NEXT_STEPS.md`).
- **Customer #2 onboarding.** When a second customer arrives, the integration shape is copied from vanguard's HTTP-only consumption. Different workstream.
- **Legal Tier 1 publisher posture.** Outrider's regulatory question, not vanguard's execution question.
- **Microsecond latency.** Wrong reference for vanguard's problem space. Don't optimize for it.

---

## Success criteria

After Year 1 ships, vanguard has:

1. Every trade replayable from outbox events + snapshots (Pillar 6).
2. Every risk-gate decision provably consistent with documented invariants (Pillar 7).
3. Every fill attributed against forecast within 24h (Pillar 8).
4. Dashboard showing per-strategy P&L / hit rate / Sharpe in real time (Pillar 9).
5. Audit trail surviving regulatory examination (Pillar 10).
6. Cross-venue arb capability when polymarket coverage ships (Pillar 4).
7. Paper-to-live promotion is a codified, audited process (Pillar 12).

When all seven hold, vanguard is operationally what AQR / DE Shaw's execution arm is to their research arm. The architectural shape that lets a research firm scale beyond personal capital.
