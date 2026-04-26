---
Status: active
Written: 2026-04-25
Author: claude-code (drafted at human direction — "i want to do a full untangling where each repo is in a fantastic place on their own only communicating cleanly through their api")
Goal-hash: (recompute with cortex doctor)
Updated-by:
  - 2026-04-25T21:00 claude-code (created)
  - 2026-04-25T21:45 claude-code (rewrote per human reframing — "the API bridge carries the data; both sides run their own loops on top of that data". Added architecture diagram, sharpened operational-state strategy from "vanguard implements its own near-clone" to "vanguard builds minimum-viable trade-execution platform", filed vanguard-execution-flywheel as a deferred follow-up plan rather than treating it as part of this work.)
  - 2026-04-25T22:00 claude-code (Status proposed → active — human green-light "let's fucking go brother we got this!!!!". Wave 1 of execution kicked off in parallel: A.2.1 reverse leak fix + B.4 vendor utilities + B.5a contract data-type mirrors + Explore audit of outrider scheduler for A.1 design.)
  - 2026-04-25T23:30 claude-code (Major reframe per human direction — "outrider does research and passes a clean and elegant API with the goal of highlighting an opportunity for a trade. Vanguard takes that information and decides what to do with it." THE PROPOSAL IS THE PRODUCT. Outrider's API surface collapses to 4 endpoint families (proposals, outcomes, strategies, reference/fees as convenience). Internal analysis (vol_score, OQS, reasoning, Shapley, autolab) stays inside outrider's pipeline; vanguard imports of that machinery become deletions, not HTTP-switches. Proposal payload extends with insight-only fields (probability_ci, conviction, edge_vs_consensus, time_horizon, agent_consensus, event_shape) PLUS a `candidate_instruments` block exposing up to 3 expression channels per proposal: kalshi, polymarket, options. Each channel is optional; vanguard picks zero/one/multiple. Trading-profile endpoint (B.1.2) deemed wrong-shaped — vanguard owns its deployment config; flagged for deprecation. Stage B reframed as "outrider API hardening," Stage C as "vanguard adapts to consume new API." New deferred plans: polymarket-coverage, options-structuring, OutriderExecution premium product, trading-profile-endpoint-deprecation.)
  - 2026-04-26T00:25 claude-code (Bar elevated per human direction — "outrider needs to be excellent and super competitive." Decoupling is the foundation; product excellence is what gets shipped on top. Added "What competitive looks like" subsection articulating the 7 dimensions outrider must be top-tier on (calibration quality, signal differentiation, coverage breadth, track-record transparency, latency, API polish, self-sufficient proposals). 9 new follow-up workstreams filed: polymarket-coverage [extended], options-structuring [extended], agents-endpoint, streaming-api, developer-experience, observability-sla, thesis-long-generation, strategy-id-population, conviction-calibration-curve. Each independently shippable post-decouple. Legal research memo confirmed Tier 1 publisher posture is achievable; architecture is already aligned; legal/regulatory follow-ups parked for post-build. Tonight's work continues uninterrupted.)
Cites: outrider/docs/CONTRACT.md, outrider/docs/SPLIT_INVENTORY.md § 1, outrider/docs/THESIS.md § Invariants, autumn-garage/.cortex/state.md
---

# Full vanguard ↔ outrider separation — outrider as polished product, vanguard as canonical customer

> **End-state: outrider is a self-contained intelligence product. Its API exposes proposals (the unique research output), reference data, and consumes outcomes — nothing else. Each proposal is a self-sufficient information bundle: edge, conviction, thesis, signal attribution, plus up to 3 candidate expression channels (kalshi, polymarket, options). Vanguard is one customer — it pulls proposals, picks expression(s), executes, posts outcomes. Zero Python-level coupling in either direction. Vanguard becomes the canonical example of how a customer integrates against outrider — copyable shape for customer #2.**

## Why (grounding)

**Outrider's product is the proposal. Everything else is the kitchen.**

Outrider does research — collectors gather signals, domain agents produce probability estimates, council synthesizes, calibration tracks per-agent realization, autolab promotes/demotes models, flywheel learns from outcomes. ALL of that happens inside outrider's pipeline. The customer-facing API surfaces only the **final outputs** of that machinery, packaged as self-sufficient `ResearchProposal` objects.

Vanguard is one customer. Its job: pull proposals, decide what to do with each (gate, size, route, hedge, monitor), report outcomes back. **The proposal is the only intelligence vanguard receives**; everything vanguard needs to make a trade decision is on the payload (or in slow-moving reference data fetched separately).

This means:

- **Outrider's API has 4 endpoint families** (`/v1/proposals`, `/v1/outcomes`, `/v1/strategies/{id}`, `/v1/reference/fees`) — that's the entire surface. Plus optional/future: `/v1/agents` (track records, v2), `/v1/intel/reasoning` (live LLM, if exit-time reasoning becomes a real product need).
- **The proposal payload is rich and self-sufficient** — insight fields (`predicted_probability`, `ci`, `conviction`, `edge_vs_consensus`, `thesis`, `signal_attribution`, `agent_consensus`, `time_horizon`, `event_shape`) + `candidate_instruments` (up to 3 optional expression channels: kalshi, polymarket, options). Vanguard's gating + sizing decisions read from this payload; no per-proposal calls to outrider for vol_score / OQS / reasoning / etc. — those are internal analysis that already shaped the proposal's edge.
- **Internal analysis stays internal.** `vol_score`, `compute_oqs`, `reason_deep`, `Flywheel.log_trade`, `ForensicReviewer`, `AgentCalibrationPipeline`, `PriceFeedManager` — outrider's machinery, untouched in outrider's codebase. Vanguard's redundant in-process imports of these get **deleted**, not HTTP-switched. The proposal already encodes their results.
- **Each product owns its own platform** — own DB, own kill switch, own alerts, own healing, own heartbeat — scoped to its own concerns. Outrider's "kill switch" halts research output if calibration breaks. Vanguard's "kill switch" halts trading if fills go bad. Different signals, different actions.

`outrider/docs/CONTRACT.md` codified this in spirit with the hard rule: *"Every call that crosses from the Research System into the Trading System (or any other consumer) goes through this API. Every outcome report that feeds back into the Research System's learning loops goes through this API. No exceptions, no backdoor DB reads, no direct in-process function imports across the boundary."*

The current codebase violates that contract at scale because the post-split work parked shared infrastructure inside outrider rather than separating it. As of 2026-04-25:

- **Vanguard imports outrider in 34 non-test source files, 195 import sites, 24 distinct outrider modules.** (`grep -rn "from outrider\|import outrider" vanguard/`)
- **Outrider imports vanguard once** — a reverse leak in `outrider/_platform/config_loader.py:647` (`from vanguard.router.brokers.kalshi import KalshiClient`).
- **15 vanguard test files** import outrider directly, including `test_resolution_engine_dogfood.py` which boots outrider's FastAPI app in-process via `ASGITransport`.
- **`vanguard/conductor/scheduler.py`** drives `outrider.autolab.*` (`runner`, `shadow_runner`, `promoter`, `meta_optimizer`) — research-domain experiment orchestration running from the trading scheduler. Wrong repo entirely.

`outrider/docs/SPLIT_INVENTORY.md § 1` identified this pattern (16 `platform_pkg/ → (trading|research)` crossings flagged "all must move") but the post-split work parked the shared code under `outrider/_platform/` and shipped, leaving vanguard pip-installing outrider as a Python package. The naming made it look like research's private platform; really it was monorepo-era shared infrastructure that never got separately re-homed.

Today's consequence: vanguard cannot bootstrap, test, or deploy without outrider being importable. CI needs an `OUTRIDER_PAT` repo secret. Railway needs `GITHUB_PAT`. Local dev needs an editable-install hack against `~/Repos/outrider`. Every leak is a coupling that any future external customer could not replicate.

The user's framing on 2026-04-25: *"these are two different apps."* Outrider is the product; vanguard is one of many possible customers. The plan below makes the architecture match that framing.

## End state

### Architecture

```
┌────────────────────────────────────────────────────────────────────────────┐
│                              OUTRIDER                                       │
│                       (the product, the brand)                              │
│                                                                             │
│   Domain: produces calibrated probability proposals from public signals     │
│                                                                             │
│   forge/  ──▶  agents/  ──▶  council/  ──▶  api/                            │
│   signals     per-domain     synthesis      HTTP server (FastAPI):          │
│               models                        - ResearchProposal              │
│                                              - TradeOutcome ingest          │
│                                              - reference data               │
│                                                (fees, trading-profile,      │
│                                                 strategies, agents)         │
│                                                                             │
│   Outrider's own learning loop                                              │
│   (consumes TradeOutcomes from ANY customer)                                │
│     flywheel/  ──▶  learning/calibration/  ──▶  autolab/                    │
│     Shapley         per-agent, per-bin,         experiments,                │
│     + Thompson      per-model-version           promote/demote agents       │
│                                                                             │
│   Outrider's own platform: own DB, own kill switch (halt research output    │
│   if calibration breaks), own alerts (collector freshness, ingest backlog,  │
│   agent timeouts), own healing, own heartbeat                               │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   │
            ╔══════════════════════│══════════════════════════════════╗
            ║         HTTP API BRIDGE — versioned, authed              ║
            ║                                                          ║
            ║   outrider → customer   |   customer → outrider          ║
            ║   ResearchProposal      |   TradeOutcome                 ║
            ║   reference data:       |                                ║
            ║   - GET /v1/reference/fees                               ║
            ║   - GET /v1/reference/trading-profile                    ║
            ║   - GET /v1/strategies/{id}                              ║
            ║   - GET /v1/agents                                       ║
            ║                                                          ║
            ║   Same shape any customer sees. Vanguard is the          ║
            ║   canonical example app, not a special insider.          ║
            ╚══════════════════════│═══════════════════════════════════╝
                                   │
┌──────────────────────────────────┴──────────────────────────────────────────┐
│                              VANGUARD                                       │
│              (one customer; example app for the next customer)              │
│                                                                             │
│   conductor/                                                                │
│     http_proposal_consumer  ◀── pulls ResearchProposals (cursor, dedup)     │
│     outbox_relay            ──▶ posts TradeOutcomes (retry, dead-letter)    │
│                                                                             │
│   vault/  ──▶  router/  ──▶  garrison/                                      │
│   risk        broker          exit eval, position monitor, trade resolver   │
│   gates,      adapters                                                      │
│   sizing,     (Kalshi,            ──▶ real markets, fills, P&L              │
│   KILL        Schwab)                                                       │
│                                                                             │
│   Vanguard's OWN learning loop (FUTURE — see deferred plan)                 │
│   (consumes vanguard's own execution events)                                │
│     execution flywheel  ──▶  realization metrics  ──▶  exec-side autolab    │
│     proposal▶trade▶fill     per-broker fill quality,    promote/demote      │
│     ▶exit▶realized PnL       per-strategy realization,   sizers, exits,     │
│                              paper/live drift            broker routes      │
│                                                                             │
│   Vanguard's own platform: own DB, own minimum-viable kill switch (halt     │
│   trading on consecutive fill failures), own minimum-viable alerts (broker  │
│   health, paper/live drift, KILL_SWITCH false-positive rate)                │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Properties

1. **The HTTP API is the entire interface.** Same shape for vanguard, same shape for the next customer.
2. **Each side has its own learning loop** observing its own signals. Outrider learns "which agent's predictions held up." Vanguard learns "which broker / sizer / exit strategy actually captured the predicted edge."
3. **Each side has its own operational platform** scoped to its concerns. The shapes echo each other; the signals and actions are completely different.
4. **Reference data flows over HTTP** with versioning, not Python imports. Outrider can rotate a fee schedule without forcing a vanguard redeploy.
5. **Zero Python coupling.** No `pip install outrider` in vanguard's pyproject. No `from outrider` in vanguard's source. No `from vanguard` in outrider's source.

### Outrider's API surface (the entire customer-facing product)

| Endpoint | Purpose | Why it's outrider's |
|----------|---------|---------------------|
| `GET /v1/proposals` (+ `{id}`) | Stream of trade-opportunity intelligence | The product itself — outrider's research output |
| `POST /v1/outcomes` | Customer reports back what happened | Outrider's flywheel learns from realization |
| `GET /v1/strategies/{id}` | Sizing recommendations (Kelly, edge thresholds, exit profiles) | Autolab-tuned from calibration data — research output |
| `GET /v1/reference/fees` | Per-venue fee schedules | Convenience: keeps outrider's edge calc and customer's P&L calc using the same numbers; not unique research IP |
| `GET /v1/agents` *(v2/future)* | Agent track records | Customer-side filtering/weighting |
| `POST /v1/intel/reasoning` *(open question)* | Live LLM intel for exit-time decisions | Only if exit-time reasoning is a real product need |

**Endpoints that should NOT exist:**
- `GET /v1/reference/trading-profile` (B.1.2 — wrong-shaped; vanguard owns its deployment config; flagged for deprecation)

### The proposal payload (self-sufficient bundle)

```
ResearchProposal {
  // Identity & versioning
  proposal_id, event_id, generated_at, valid_until,
  research_version, model_version,

  // Source
  agent_name,
  agent_consensus: {agreeing_agents, disagreeing_agents, consensus_score},

  // Event description (instrument-agnostic)
  event_description,                 // "FOMC Dec 2026: 25bps cut?"
  event_shape: "binary"|"continuous"|"discrete"|"sequential",
  cohort_starts: [string],

  // Probability output (the differentiated piece)
  predicted_probability,             // outrider's point estimate
  predicted_probability_ci: {low, high},
  conviction,

  // Edge (vs market consensus where outrider has visibility)
  market_consensus_probability,
  market_consensus_source,           // "kalshi:FED-DEC-25BP" | "fed_funds_futures_imply" | ...
  edge_vs_consensus,
  edge_tier,                         // "low"|"medium"|"high"

  // Time horizon
  time_horizon: {window_start, window_end, days_to_resolution},

  // Why
  thesis_summary,
  thesis_long,
  signal_attribution: [{source, weight, contribution}, ...],

  // Linkage to sizing knobs
  strategy_id,

  // Candidate expression channels — up to 3, all optional
  candidate_instruments: {
    "kalshi":     null | { venue, ticker, side, market_price, edge_at_instrument, liquidity },
    "polymarket": null | { venue, market_slug, side, market_price, edge_at_instrument, liquidity },
    "options":    null | { venue, underlying, structure, legs, indicative_net_debit, payoff_shape, max_loss, liquidity }
  }
}
```

Vanguard's logic on receipt:

1. Gate the insight (edge ≥ min, conviction ≥ min, cohort caps, KILL_SWITCH, etc.)
2. Pick instrument(s) — one, multiple (hedge), or none
3. Size + route + monitor
4. POST outcome at every state change

### Repo shape after the work

```
vanguard/                   outrider/
├── pyproject.toml          ├── pyproject.toml
│   (no outrider dep)       │   (no vanguard dep)
├── vanguard/               ├── outrider/
│   (no `from outrider.*`)  │   (no `from vanguard.*`)
├── tests/                  ├── tests/
│   (HTTP-mocked outrider)  │   (no vanguard fixtures)
├── railway.json            ├── railway.json
│   (no GITHUB_PAT step)    │   (unchanged)
└── .github/workflows/      └── .github/workflows/
    (no sibling checkout,       (unchanged)
     no OUTRIDER_PAT)
```

Local bootstrap, both repos: `uv venv && uv pip install -r requirements.txt && pytest`. No PATs, no sibling repos, no editable installs. AST cluster-boundary tests fail any future `from outrider` in vanguard or `from vanguard` in outrider.

## Approach

The 24 outrider modules vanguard imports decompose into five categories, each with a different separation strategy:

| Category | Modules | Strategy |
|----------|---------|----------|
| **Reference data** | `outrider._platform.fees`, `outrider._platform.trade_math`, `outrider._platform.trading_profile`, `outrider._platform.agent_registry` | Fetch via HTTP. Outrider grows endpoints; vanguard cache-on-pull. |
| **Stable infra utilities** | `outrider._platform.atomic_write`, `outrider._platform.paths`, `outrider._platform.git_sha`, `outrider._platform.exceptions`, `outrider._platform.broker_exceptions`, `outrider._platform.api_keys` | Vendor into vanguard. ~50–200 LOC each, no behavior. |
| **Database** | `outrider._platform.db`, `outrider._platform.db_queries` | Vanguard owns its own SQLAlchemy schema for tables it owns (events_outbox, outcomes_inbox, vanguard_proposal_cursors, risk_state). Vanguard's DB is already separate at the Postgres level (`vanguard-db` per state.md); just port the models. |
| **Operational state** | `outrider._platform.kill_switch`, `outrider._platform.state_store`, `outrider._platform.config_loader`, `outrider._platform.watchtower.*` (alerts, auto_halt), `outrider._platform.lorien.*` (health, healing, heartbeat) | Vanguard builds a **minimum-viable trade-execution platform**: own DB (B2 above), own basic kill switch, own basic alerts, own basic config loader. This is *not* "port outrider's machinery." Outrider's lorien/watchtower were designed for research-pipeline health (collector freshness, ingest backlog, agent timeouts); vanguard's signals are different (broker fail rates, paper/live drift, KILL_SWITCH false-positives, fill timeline anomalies). Vanguard's first version is small — possibly just Railway's `restartPolicyType: ON_FAILURE` plus a basic alerts hook. **Lorien-style healing and watchtower-style multi-axis monitoring are deferred to vanguard's own learning-loop plan** (see follow-ups). The decoupling unblocks that plan; it does not require it. |
| **Typed contract** | `outrider.api.Event`, `outrider.api.StrategyParams`, `outrider.api.get_params` | Mirror in vanguard as pydantic models that match the HTTP JSON shape. Outrider serves the JSON; vanguard parses into its own types. Standard customer pattern. |
| **Research-domain logic in wrong repo** | `outrider.autolab.runner`, `outrider.autolab.shadow_runner`, `outrider.autolab.promoter`, `outrider.autolab.meta_optimizer` | Delete from vanguard. Move orchestration to outrider's scheduler. Vanguard pulls active strategy params via API. |

The migration runs in four stages. Stages A and D are mandatory bookends; B and C are the bulk of the work.

### Stage A — cleanup / preconditions

**A1. Delete `outrider.autolab.*` from vanguard's scheduler.** Move the `run_experiment_loop` / `shadow_runner` / `promote_candidate` / `meta_optimizer` invocations into outrider's own scheduler (or wherever in outrider they belong). Vanguard's `vanguard/conductor/scheduler.py` ends up shorter and stops driving research operations entirely. **PR on vanguard + PR on outrider, sequenced (outrider first, so the autolab loop keeps running uninterrupted).**

**A2. Fix the reverse leak.** `outrider/_platform/config_loader.py:647` stops importing `from vanguard.router.brokers.kalshi`. Move broker config validation to vanguard's startup (vanguard's config validates its own broker creds). Outrider's config_loader becomes vanguard-agnostic. **Single PR on outrider, then a small PR on vanguard.**

### Stage B — build-out (largely parallel)

**B1. Outrider exposes reference-data endpoints.** New endpoints (or extensions to existing ones) for vanguard (and any future customer) to fetch:

- `GET /v1/reference/fees` — fee schedules per venue (Kalshi, Schwab options)
- `GET /v1/reference/trading-profile` — trading profile shape (cluster, env)
- `GET /v1/strategies/{strategy_id}` — active strategy params (already partially exists per `outrider.api.StrategyParams`/`get_params`; verify shape and document)
- `GET /v1/agents` — agent roster (already exists per `test_outrider_agents_endpoint.py`)

Each endpoint comes with contract-conformance tests under `outrider/tests/aegis/unit/test_outrider_*_endpoint.py`. **PR on outrider per endpoint or grouped (3–5 PRs).**

**B2. Vanguard ports its DB models.** Create `vanguard/_platform/db.py` with SQLAlchemy models for the tables vanguard owns (`vanguard_events_outbox`, `vanguard_outcomes_inbox`, `vanguard_proposal_cursors`, `vanguard_risk_state`, etc.). Migration: dual-write under both old and new model names for one deploy, verify the new tables, switch reads. **2–3 PRs on vanguard, sequential.**

**B3. Vanguard builds minimum-viable operational platform.** New modules under `vanguard/_platform/`: `kill_switch.py`, `state_store.py`, `config_loader.py`, `alerts.py`. Scope is deliberately small — vanguard's first version is *the minimum that lets vanguard run on Railway without outrider's machinery*, not a port of outrider's lorien/watchtower stack. Things to ship:
- `kill_switch.py` — a simple env-var or DB-backed flag, checked at every entry path. Halts new orders. ~50 LOC.
- `state_store.py` — wraps StateStore writes against vanguard's own DB. ~100 LOC.
- `config_loader.py` — pydantic-settings reading from env, no fancy validation cascades. ~100 LOC.
- `alerts.py` — a thin wrapper over the existing alert channel (Slack webhook or whatever vanguard uses today). ~50 LOC.

Things explicitly NOT to ship in this stage: `auto_halt`, `health`, `healing`, `heartbeat`. These were research-flavored multi-axis monitoring; vanguard needs its own version of the *concept* but with trade-execution signals (broker fail rate, paper/live drift, fill timeline). That work is the domain of the **vanguard-execution-flywheel** follow-up plan — not this one. The decoupling unblocks it; it does not require it.

For any call site in vanguard's source that today imports `outrider._platform.lorien.*` or `outrider._platform.watchtower.*`, replace with: a no-op stub, a Railway-native equivalent (`restartPolicyType: ON_FAILURE` for healing), or a TODO referencing the follow-up plan. Vanguard runs fine on Railway with much less monitoring than it currently borrows from outrider; the borrowed machinery isn't load-bearing in a way that requires same-day replacement. **2–3 PRs on vanguard.**

**B4. Vanguard vendors stable utilities.** Copy `atomic_write.py`, `paths.py`, `git_sha.py`, `exceptions.py`, `broker_exceptions.py`, `api_keys.py` into `vanguard/_platform/`. Update imports. **One PR on vanguard.**

**B5. Vanguard mirrors `outrider.api` types as pydantic models.** Create `vanguard/contract/models.py` with `ResearchProposal`, `TradeOutcome`, `Event`, `StrategyParams` mirroring the HTTP JSON shapes. Vanguard's HTTP consumers serialize/deserialize via these. **One PR on vanguard.** Versioned to match outrider's contract version.

### Stage C — cutover

**C1. Vanguard switches reference-data calls to HTTP.** `vanguard/_platform/fees.py` becomes a thin client of `GET /v1/reference/fees` with a startup-cache. Same pattern for `trade_math`, `trading_profile`. Depends on B1 and B5. **One PR on vanguard.**

**C2. Vanguard tests switch to HTTP mocks.** Replace every `from outrider.api.server import create_app` and ASGITransport pattern with `respx`-mocked HTTP responses. The 15 test files importing outrider become 0. `test_resolution_engine_dogfood.py` either gets HTTP-mocked or moves to outrider as a contract-conformance test. Depends on B1, B5. **2–3 PRs on vanguard.**

**C3. Drop the dependency.** Remove `outrider @ git+https://...` from vanguard `pyproject.toml`. Remove the `GITHUB_PAT` block from `vanguard/railway.json` buildCommand. Remove the sibling-checkout step + `OUTRIDER_PAT` reference from `vanguard/.github/workflows/test.yml`. Verify Railway deploy and CI both pass without any outrider artifact present. **One PR on vanguard. This is the symbolic ship.**

### Stage D — enforcement

**D1. AST cluster-boundary tests, both directions.**
- `vanguard/tests/aegis/unit/test_no_outrider_imports.py` — fails if any `vanguard/**.py` (excluding `vanguard/contract/`, which mirrors types by hand) imports `outrider.*`.
- `outrider/tests/aegis/unit/test_no_vanguard_imports.py` — fails if any `outrider/**.py` imports `vanguard.*`.

Both run in CI as part of the core suite. **One PR each.**

**D2. Final verification + journal entry.** Fresh clone of each repo on a new machine, run `uv venv && uv pip install -r requirements.txt && pytest`. Both green. Local-dev `setup.sh` simplified to remove sibling-detection logic. Write `journal/2026-XX-XX-vanguard-outrider-fully-separated.md` documenting the journey, citing this plan and `SPLIT_INVENTORY.md` § 1 as the work that finally landed. **One PR per repo + one Journal entry on autumn-garage.**

## Success Criteria

1. **Zero `from outrider` imports in `vanguard/` source** (verified by `grep -rn "from outrider\|import outrider" vanguard/ --include="*.py" | grep -v vanguard/contract/` returning empty). Cluster-boundary AST test enforces this on every PR.
2. **Zero `from vanguard` imports in `outrider/` source** (verified by `grep -rn "from vanguard\|import vanguard" outrider/ --include="*.py"` returning empty). Cluster-boundary AST test enforces this on every PR.
3. **Vanguard `pyproject.toml` lists no outrider dep** — `grep -i outrider vanguard/pyproject.toml` returns empty.
4. **Vanguard `railway.json` buildCommand contains no `GITHUB_PAT`** — `grep GITHUB_PAT vanguard/railway.json` returns empty.
5. **Vanguard `.github/workflows/test.yml` contains no `OUTRIDER_PAT` and no sibling-checkout step** — `grep -i 'outrider\|OUTRIDER_PAT' vanguard/.github/workflows/test.yml` returns empty.
6. **Fresh-clone bootstrap works on both repos** — on a machine with no other repos cloned: `git clone <repo> && cd <repo> && uv venv && uv pip install -r requirements.txt && bash scripts/test-trade-critical.sh` exits 0. Verified on a fresh CI runner via the existing `.github/workflows/test.yml` workflow.
7. **Outrider exposes the reference-data endpoints** — `GET /v1/reference/fees`, `GET /v1/reference/trading-profile`, `GET /v1/strategies/{id}` documented in `outrider/docs/CONTRACT.md` and tested in `outrider/tests/aegis/unit/test_outrider_*_endpoint.py`.
8. **No regressions in existing CI core suites** — outrider's 18-file core (166 tests) and vanguard's 20-file core (567 tests) stay green throughout. Each phase's PR runs through the existing gate.

## Work items

### Stage A (cleanup, ~1 week)
- [ ] A1.1 (outrider): move `autolab.runner`/`shadow_runner`/`promoter`/`meta_optimizer` driving from vanguard's scheduler into outrider's own scheduler
- [ ] A1.2 (vanguard): delete autolab call sites from `vanguard/conductor/scheduler.py`; verify experiments still run end-to-end on Railway
- [ ] A2.1 (outrider): rewrite `outrider/_platform/config_loader.py:647` to not import from vanguard
- [ ] A2.2 (vanguard): move broker-config validation to vanguard's startup

### Stage B (build-out, ~2 weeks, mostly parallel)
- [ ] B1.1 (outrider): document + ship `GET /v1/reference/fees` endpoint + contract test
- [ ] B1.2 (outrider): document + ship `GET /v1/reference/trading-profile` endpoint + contract test
- [ ] B1.3 (outrider): verify/document existing `GET /v1/strategies/{id}` endpoint matches what `outrider.api.get_params` returns
- [ ] B2.1 (vanguard): port `events_outbox`, `outcomes_inbox`, `vanguard_proposal_cursors`, `vanguard_risk_state` SQLAlchemy models to `vanguard/_platform/db.py`
- [ ] B2.2 (vanguard): dual-write deploy + verify + cutover reads
- [ ] B3.1 (vanguard): build minimum-viable `kill_switch.py`, `state_store.py`, `config_loader.py`, `alerts.py` in `vanguard/_platform/`
- [ ] B3.2 (vanguard): replace `lorien.*` and `watchtower.auto_halt` call sites with no-op stubs / Railway-native equivalents / TODO-references to vanguard-execution-flywheel plan
- [ ] B4 (vanguard): vendor `atomic_write`, `paths`, `git_sha`, `exceptions`, `broker_exceptions`, `api_keys` into `vanguard/_platform/`
- [ ] B5 (vanguard): create `vanguard/contract/models.py` mirroring `outrider.api` types as pydantic models, version-pinned to outrider's contract version

### Stage C (cutover, ~1 week)
- [ ] C1 (vanguard): switch `vanguard/_platform/fees.py` and `trade_math.py` and `trading_profile.py` to HTTP-client wrappers, with startup cache
- [ ] C2.1 (vanguard): replace `outrider.api.server.create_app()` ASGITransport patterns in tests with `respx` HTTP mocks
- [ ] C2.2 (vanguard): move or rewrite `test_resolution_engine_dogfood.py` (HTTP-mocked here, or contract-conformance in outrider)
- [ ] C2.3 (vanguard): rewrite remaining 14 test files that import outrider
- [ ] C3.1 (vanguard): drop `outrider @ git+https://...` from `pyproject.toml`
- [ ] C3.2 (vanguard): simplify `railway.json` buildCommand (no `GITHUB_PAT` step)
- [ ] C3.3 (vanguard): simplify `.github/workflows/test.yml` (no sibling checkout, no `OUTRIDER_PAT`)
- [ ] C3.4 (vanguard): verify Railway deploy + CI both green from a fresh clone

### Stage D (enforcement, ~2 days)
- [ ] D1.1 (vanguard): AST test that fails on any `from outrider` in `vanguard/**.py` (excluding `vanguard/contract/`)
- [ ] D1.2 (outrider): AST test that fails on any `from vanguard` in `outrider/**.py`
- [ ] D2.1: fresh-clone verification on a clean machine, both repos
- [ ] D2.2 (autumn-garage): journal entry citing this plan; status flips to `shipped`

## What competitive looks like

The decoupling is the foundation; **product excellence is what ships on top**. Seven dimensions outrider must be top-tier on to be a genuinely competitive intelligence product (vs. a clean-but-mediocre API):

1. **Calibration quality** — per-agent Brier scores, sharper CIs, demonstrably better than market consensus over time. The product IS the calibration; bad calibration = bad customer outcomes regardless of how nice the API is.
2. **Signal differentiation** — unique signals competitors don't have. The Renaissance-shape moat (public inputs, superior processing). Each agent's `signal_attribution` exposes sources customers can't replicate alone.
3. **Coverage breadth** — kalshi + polymarket + options + (later) futures + equity + FX. Kalshi-only is the narrowest possible product; the polymarket and options coverage workstreams are non-negotiable for v1 maturity.
4. **Track-record transparency** — `/v1/agents/{name}` exposes per-agent calibration with model_version segmentation. Customers filter "show me proposals from agents with Brier < 0.18 over last 90d, conviction > 0.7."
5. **Latency** — proposals issued before market consensus moves. Push channel (WebSocket / SSE) for real-time customers, not just GET-poll. Time-to-emission is the moat.
6. **API polish** — OpenAPI spec, Python SDK, examples site, status page, p99 latency public, versioned schema documentation, change-log per `research_version`.
7. **Self-sufficient proposals** — Shape 3 already points right here. Each proposal stands alone; customer reads it and has everything to act. Don't backslide into "fetch X to interpret field Y."

The decoupling work in Stages A-D is the foundation. The 9 follow-up workstreams below ship on that foundation to get to "excellent and super competitive."

## Follow-ups (deferred)

### Excellence workstreams (post-decouple, independently shippable)

- **Polymarket coverage** — populate `candidate_instruments.polymarket`. Outrider's `forge/` needs a polymarket collector + market roster mapping. Without this, outrider is kalshi-only; with it, outrider becomes the canonical multi-venue intelligence product. Resolved to: `plans/outrider-polymarket-coverage.md` (TBC).
- **Options structuring** — populate `candidate_instruments.options`. Council-side logic that maps insights to specific options structures (calendar spreads, vol plays, directional spreads). Requires options-chain awareness in `forge/`. Resolved to: `plans/outrider-options-structuring.md` (TBC).
- **`/v1/agents/{name}` track-record endpoint** — per-agent calibration metrics (Brier score, hit rate, recent realization, model_version-segmented). Customer-side filtering/risk-weighting. Frame as "model performance research" with methodology disclosure (legal Tier 1 hygiene). Resolved to: `plans/outrider-agents-endpoint.md` (TBC).
- **Streaming / push API** — WebSocket or SSE channel pushing proposals as they're emitted. Time-to-emission is the moat; customers polling on cadence lose to customers receiving instant notification. Resolved to: `plans/outrider-streaming-api.md` (TBC).
- **Developer experience** — OpenAPI spec auto-generated from outrider's API; Python SDK; examples site (Postman / curl / SDK quickstarts); change-log per `research_version`. Resolved to: `plans/outrider-developer-experience.md` (TBC).
- **Observability + SLA** — public status page, p99/p999 latency dashboards, calibration drift alerts (per-agent Brier trending up triggers internal investigation), uptime measurement. Resolved to: `plans/outrider-observability-sla.md` (TBC).
- **`thesis_long` generation** — currently null on the wire. Council-side long-form synthesis writes the thesis narrative for each proposal. Customers want the audit trail / human-readable why. Resolved to: `plans/outrider-thesis-long-generation.md` (TBC).
- **`strategy_id` population** — currently null. Each agent emits which strategy bucket the proposal belongs to. Wires up the customer's `/v1/strategies/{id}` lookup. Resolved to: `plans/outrider-strategy-id-population.md` (TBC).
- **Conviction calibration curve** — empirical mapping from `conviction` value to realization rate, exposed to customers. "0.85 conviction empirically realizes Y% of expected edge" — customers can size accordingly. Differentiator vs. competitors who emit conviction without auditable calibration. Resolved to: `plans/outrider-conviction-calibration-curve.md` (TBC).
- **Layer 2 invariant test coverage** — proactive tests for invariants articulated in vanguard/outrider CLAUDE.md that don't yet have direct coverage. Vanguard side: 14 risk gates parameterized, KILL_SWITCH-on-every-entry-path property, paper/live convergence property, outbox-relay retry-with-backoff, cursor-fails-closed-on-5xx. Outrider side: per-bin calibration drift assertions, Shapley sum-to-1, agent-base contract conformance. Multi-week workstream; AGENTS.md "every fix gets a regression test" rule grows it organically too. Resolved to: `plans/layer-2-invariant-coverage.md` (TBC).
- **Per-customer auth + rate limiting** — multi-customer-ready API key issuance, revocation, per-customer rate limits, per-customer usage analytics. Pre-launch requirement once customer #2 onboards. Builds on observability + SLA. Resolved to: `plans/outrider-per-customer-auth.md` (TBC).
- **Customer onboarding flow** — when customer #2 lands: API key issuance, ToS acceptance, billing, integration docs. Pairs with developer experience + per-customer auth. Out of scope until first external customer is real. Resolved to: `plans/outrider-customer-onboarding.md` (TBC).

### Other deferred

- **OutriderExecution (premium product layer).** Given an insight + a customer's broker capabilities + capital constraints, recommend the optimal cross-channel expression (kalshi + options + polymarket combination for THIS customer). Customer-specific; NOT in OutriderResearch's core. Likely separately-priced product line. Resolved to: `plans/outrider-execution-product.md` (TBC).
- **Trading-profile endpoint deprecation.** B.1.2 shipped `GET /v1/reference/trading-profile` but the sharper architectural review concluded vanguard's deployment config doesn't belong on outrider's API surface. Vanguard reads trading-profile from its own env/config. The endpoint stays dormant until cleanup. Resolved to: `plans/deprecate-trading-profile-endpoint.md` (TBC).
- **Legal/regulatory pre-launch checklist.** Tier 1 publisher posture is the simplest defensible path per the legal research memo. Architecture is already aligned. Pre-launch: counsel review of customer agreement, ToS / marketing copy, state IA registration survey, disinterestedness audit, track-record framing review. Resolved to: `plans/outrider-legal-tier1-compliance.md` (TBC). Architecture work continues uninterrupted; legal review happens before first paying external customer.
- **Vanguard's own execution flywheel + autolab.** Vanguard currently has no learning loop of its own — it's been free-riding on outrider's flywheel/lorien/watchtower machinery for orchestration. Post-separation, vanguard needs to design its own answer to: "which broker / sizer / exit strategy / risk gate actually captured the predicted edge?" That includes a per-trade flywheel (proposal → trade → fill → exit → realized vs predicted PnL), realization metrics per-broker / per-strategy / per-paper-live-mode, an exec-side autolab that promotes/demotes sizers and exits and broker routes, and the multi-axis monitoring (broker health, paper/live drift, KILL_SWITCH false-positive rate) that lorien-style healing and watchtower-style alerts would protect. **Resolved to:** future `plans/vanguard-execution-flywheel.md` (TBC). Out of scope here; the decoupling unblocks it.
- **Audit outrider's own test suite for trade-side leaks.** `outrider/tests/aegis/integration/{paper_trade_pipeline,trade_lifecycle_canary,options_paper_pipeline,unified_routing}.py` are vanguard-domain tests living in outrider. Move to vanguard or delete. Resolved to: `journal/2026-04-25-ci-test-bring-up.md` (filed during Layer-1 CI work) → follow-up plan to be created when this plan reaches Stage D.
- **Outrider's six collection-error test files** (`test_calibration.py`, `test_weather_wiring.py`, etc.) reference renamed module paths post-split. Resolved to: `journal/2026-04-25-ci-test-bring-up.md` → out of scope here; quick fix in a separate PR.
- **Outrider cluster-registry test-isolation bug** (worked around via per-file pytest invocation in `scripts/test-trade-critical.sh`). Resolved to: `journal/2026-04-25-ci-test-bring-up.md` → out of scope here.
- **Layer-2 invariant coverage** (14 risk gates parameterized, KILL_SWITCH-on-every-entry-path property test, paper/live convergence property test, calibration drift assertions, Shapley sum-to-1) — long-running coverage workstream. Resolved to: future `plans/coverage-invariant-tests.md` (TBC).

## Known limitations at exit

- **Reference-data caching strategy is per-call-site, not centralized.** Vanguard's `vanguard/_platform/fees.py` will fetch on startup and cache in memory; if outrider updates its fee table mid-day, vanguard won't see it until the next restart. Acceptable for fees (slow-moving) but watch for issues if `trading_profile` or `strategy_params` need higher freshness. Future work: a generic "reference-data refresh loop" in vanguard that polls each endpoint at configurable intervals.
- **Contract version drift between vanguard and outrider** is now a real risk. Today, `vanguard.contract.models` is hand-mirrored from `outrider.api`; if outrider bumps `research_version` and adds a field, vanguard doesn't see it until someone updates the mirror. Mitigations: (1) outrider's `/v1/contract` endpoint returns the schema and version; vanguard's CI fetches it and diffs against its mirror; (2) add a `vanguard.contract.tests` suite that round-trips JSON fixtures recorded from outrider's actual responses. Sketch in journal post-Stage-C; not in this plan's scope.
- **No private PyPI for vanguard.** Option A from the architecture discussion (extracting `sigint-platform` as a published shared package) was deliberately rejected in favor of vendoring. If a third or fourth customer eventually appears and the vendored utilities start to genuinely diverge, revisit. Resolved to: this plan's "Why" section + the journal entry that ratifies this approach.
- **Stage A.1 requires careful sequencing on Railway** — autolab orchestration must keep running during the cutover. Stage A's plan is "ship outrider's autolab driver first, verify experiments still tick, then delete from vanguard." If outrider's deploy lags vanguard's, experiments stop. Mitigation: feature-flag the cutover with `AUTOLAB_DRIVER=outrider|vanguard` env on both repos.
