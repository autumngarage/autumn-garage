# LLM provider contract opened as Doctrine 0003; Kimi rollout follows

**Date:** 2026-04-20
**Type:** decision
**Trigger:** T1.1
**Cites:** doctrine/0003-llm-providers-compose-by-contract, plans/llm-provider-additions, doctrine/0001-why-autumn-garage-exists, doctrine/0002-interactive-by-default

> Established the cross-tool contract for adding LLM providers (env var, base URL, default model, smoke test, setup UX) so Touchstone and Sentinel can both gain Kimi without code-coupling and without UX drift. Kimi rollout plan opened as the first application of the contract.

## Context

User asked where Kimi (Moonshot AI) should land in the garage. Survey of the three tools showed:

- **Touchstone:** reviewer cascade in `.codex-review.toml` already supports a list of providers; needs a new `kimi` entry.
- **Sentinel:** `src/sentinel/providers/` is fully pluggable (claude.py, openai.py, gemini.py, local.py); needs a new `moonshot.py` and `ProviderName` enum entry.
- **Cortex:** no LLM calls today; Phase C synthesis (`refresh-map`, `refresh-state`) plans to shell out to LLM CLIs and will inherit the same provider menu.

Kimi research (Moonshot docs at `platform.kimi.ai/docs/api/quickstart.md`):

- OpenAI-compatible endpoint at `https://api.moonshot.ai/v1`, auth via `MOONSHOT_API_KEY`, default model `kimi-k2.6`.
- Official `kimi` CLI exists but is interactive-only — no `-p` flag, no stdin pipe, no JSON output. Not suitable for hook integration. Use the HTTP endpoint via `openai` SDK (Python) or `curl` (shell) instead.

The naive next step would be a shared `garage-providers` library both tools import. That violates Cortex Doctrine 0002 (no shared code between tools) and Autumn Garage Doctrine 0001 (no code dependencies between trio repos). The user explored a fourth option — a standalone "orchestration as a service" tool — and self-corrected: with two consumers and config-driven routing, it's overkill, and the symptom (UX drift) is fixable with a shared *contract* rather than shared code.

## What we decided

1. **Doctrine 0003 establishes the contract.** Five fields every provider must define identically across consuming tools: provider identity, auth env var, endpoint shape, default model, smoke test. Plus a setup UX rule that inherits Doctrine 0002 (interactive-by-default).
2. **The providers reference table lives outside doctrine** at `autumn-garage/integration/providers.md`. Doctrine names the rule; the table is mutable as providers are added.
3. **Kimi is the first application.** A new plan `plans/llm-provider-additions.md` tracks the rollout: Kimi via OpenAI-compatible HTTP, identifier `kimi`, env var `MOONSHOT_API_KEY`, default model `kimi-k2.6`, smoke test against `/v1/models`. Per-tool PRs in touchstone and sentinel land independently against the shared contract.
4. **Cortex Phase C inherits.** When Cortex ships `refresh-map` / `refresh-state`, it picks up the same provider menu without renegotiation.

Considered and rejected:

- **Shared library.** Forbidden by Doctrine 0002 + 0001. Reintroduces release-cadence coupling.
- **Standalone orchestrator tool.** Overkill for two consumers; routing is config, not runtime intelligence; LiteLLM/OpenRouter cover the same ground if it's ever needed.
- **Per-tool independent setup with no coordination.** Guaranteed to drift on env var names and prompts; bad first-time UX.

## Consequences / action items

- [ ] Create `autumn-garage/integration/providers.md` with the providers reference table seeded with current providers (claude, codex/openai, gemini, local) plus kimi.
- [ ] Open `plans/llm-provider-additions.md` (this commit) to track per-tool PRs.
- [ ] Touchstone PR: add `kimi` to reviewer cascade + corresponding hook wrapper using `MOONSHOT_API_KEY` + OpenAI-compatible HTTP.
- [ ] Sentinel PR: add `src/sentinel/providers/moonshot.py` (likely a thin subclass of `openai.py` with base URL override) + `ProviderName.MOONSHOT` entry + setup wizard prompt.
- [ ] Update `state.md` with new workstream once both PRs are open.
- [ ] After both ship, dogfood: configure Kimi in autumn-mail and run one `sentinel work` cycle with `coder = kimi`. Journal the result.
