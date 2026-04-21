# Route Conductor's Kimi adapter through Cloudflare Workers AI, not Moonshot direct

**Date:** 2026-04-21
**Type:** decision
**Trigger:** T2.1 (user-phrased decision; also touches integration/providers.md which is load-bearing but not inside .cortex/)
**Cites:** doctrine/0004-conductor-as-fourth-peer, plans/conductor-bootstrap, integration/providers.md, journal/2026-04-20-conductor-decision, https://developers.cloudflare.com/changelog/post/2026-04-20-kimi-k2-6-workers-ai/

> During Conductor v0.1 implementation, Cloudflare shipped Day 0 hosting for Kimi K2.6 on Workers AI (2026-04-20). We pivoted the Kimi adapter to call Cloudflare's OpenAI-compatible endpoint (`/client/v4/accounts/{account_id}/ai/v1/chat/completions`) instead of Moonshot's own `api.moonshot.ai`. The user's framing: *"it's good for us to invest in using cloudflare tools."* Direct Moonshot backend remains available as a future config knob on the `kimi` provider, not a new identifier.

## Context

The original Conductor bootstrap plan (`plans/conductor-bootstrap.md`) specified Kimi via `https://api.moonshot.ai/v1` with `MOONSHOT_API_KEY`. After scaffolding and shipping the adapter at that shape (PR #1 opened against autumngarage/conductor), a pre-live-test checkpoint surfaced the question: should we route via Cloudflare instead?

Cloudflare published Day 0 Kimi K2.6 support on Workers AI on 2026-04-20 — one day before this decision. Three options surfaced:

- **A — Workers AI direct.** CF hosts Kimi on their GPUs. `CLOUDFLARE_API_TOKEN` + `CLOUDFLARE_ACCOUNT_ID`. No Moonshot account needed.
- **B — AI Gateway proxying Moonshot direct.** CF sits in front of api.moonshot.ai for analytics/caching. Still needs `MOONSHOT_API_KEY`; Moonshot isn't on CF's supported-providers list so it'd use "universal endpoint" mode.
- **C — Workers AI through AI Gateway.** CF-hosted model, proxied through CF's own gateway. Best observability but most setup.

## What we decided

**Option A — Workers AI direct.** The user's reasoning: going all-in on Cloudflare as a platform investment. Cleanest setup, one credential surface that will serve future CF-hosted models too, lowest latency (edge network), one bill.

Concrete changes to Conductor v0.1 (landed on branch `feat/v0.1-scaffold-kimi-adapter`):

- `src/conductor/providers/kimi.py` rewritten to use CF endpoint and `CLOUDFLARE_API_TOKEN` + `CLOUDFLARE_ACCOUNT_ID` env vars.
- Default model changed from `kimi-k2.6` to `@cf/moonshotai/kimi-k2.6` (CF's namespaced form).
- `smoke()` simplified to a 1-token chat completion (CF's OpenAI-compat surface has no `/models` endpoint today).
- All 20 tests updated and passing.
- `integration/providers.md` kimi row updated.
- Conductor's README and CLAUDE.md updated.

**Identifier stays as `kimi`, not `kimi-cf`.** The provider identifier in Conductor names the model family; the backend (Cloudflare vs Moonshot direct) is a configuration detail. If a future consumer needs the direct Moonshot backend (e.g., for access to `kimi-k2-thinking` or `kimi-k2-turbo-preview` which weren't on Cloudflare at time of writing), that lands as a `--backend moonshot-direct` knob on the existing provider, not as a separate row in the providers table.

Considered and rejected:

- **Stay on Moonshot direct** — misses the platform-investment benefit the user called out; adds a Moonshot account to the user's setup surface; splits billing.
- **AI Gateway (Option B)** — Moonshot isn't on CF's supported-providers list; universal-endpoint mode works but doesn't get us the edge-hosting latency benefit; still requires two accounts.
- **Both backends at v0.1** — defer. Conductor v0.1 scope was already "ship Kimi end-to-end"; adding backend selection doubles the test matrix without a current consumer asking for it.

## Consequences / action items

- [x] `kimi.py` rewritten for Cloudflare backend.
- [x] Tests updated (20 passing).
- [x] `integration/providers.md` kimi row rewritten.
- [x] Conductor README + CLAUDE.md updated.
- [ ] Commit the Cloudflare-routing changes on the same PR (PR #1 — feat/v0.1-scaffold-kimi-adapter).
- [ ] Live smoke test: user provisions `CLOUDFLARE_API_TOKEN` + `CLOUDFLARE_ACCOUNT_ID`, runs `uv run conductor call --with kimi --task "ping"` against real CF, confirms behavior matches the mocked tests.
- [ ] If/when a consumer needs the direct Moonshot backend, open a Conductor plan for `--backend moonshot-direct`. Not tracked as blocking today.

## Reflection

The pivot happened because we stopped to ask "where should Kimi live?" before flipping the switch on a real API key. Good habit — a day-old Cloudflare release that materially changes the right answer wouldn't have surfaced if we'd just run the live test and shipped. The broader signal: when a v0.1 is "a week of plans and a few hours of code," the plans are robust but the code is new enough to absorb late-breaking platform shifts without architectural pain. Future Conductor work should preserve that posture — defer anything that isn't forced into v0.1.
